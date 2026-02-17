---
name: spring-boot-testing
description: Use this skill whenever generating, reviewing, or refactoring tests for Java Spring Boot applications. Enforces a three-layer testing pyramid with package-level unit tests, sliced integration tests, and black-box full integration tests. Covers BDD Mockito, AssertJ, OkHttp MockWebServer for HTTP clients, Testcontainers for databases, and strict conventions for test structure, naming, and organization. Apply alongside the spring-boot-clean-code skill.
---

# Spring Boot Testing

This skill defines the testing strategy, conventions, and patterns for Spring Boot applications that follow the **spring-boot-clean-code** skill. It enforces a **three-layer testing pyramid** with strict rules for what each layer covers and how tests are written.

-----

## Core Principle: The Testing Pyramid

Tests are organized in three layers. The bulk of coverage comes from cheap, fast unit tests. Integration tests exist to verify contracts and infrastructure. Full integration tests are a thin safety net over critical paths.

```
        /‾‾‾‾‾‾‾‾‾‾\
       / Full Integ. \          ← One per endpoint/journey. Happy path only.
      /    @SpringBoot \           Black box. Real stack + Testcontainers.
     /      Test        \          External clients mocked with @MockitoBean.
    /────────────────────\
   /   Slice Tests        \     ← @WebMvcTest for HTTP contracts.
  /    @WebMvcTest          \      @DataJpaTest for persistence.
 /     @DataJpaTest          \     Focused and fast.
/─────────────────────────────\
/        Unit Tests            \ ← Package-level. Real collaborators within module.
/  Mocks only at infrastructure \   Mocks: repositories, HTTP clients, message producers.
/   boundaries. Bulk of coverage.\  MockWebServer for HTTP client tests.
\─────────────────────────────────/
```

| Layer                  | Speed   | Context      | What It Proves                                      | Volume        |
|------------------------|---------|--------------|-----------------------------------------------------|---------------|
| **Unit**               | Fast    | No Spring    | Business logic, edge cases, error paths              | Bulk (80%+)   |
| **Slice**              | Medium  | Partial      | HTTP contracts, validation, query correctness         | Moderate (15%)|
| **Full Integration**   | Slow    | Full         | Endpoint contract + DB state, end-to-end happy path   | Few (5%)      |

-----

## Test Source Structure

Tests mirror the feature-based module structure from the main source. Each module has `unit` and `integration` sub-packages.

```
src/test/java/com/example/app/
├── order/
│   ├── unit/
│   │   ├── OrderServiceTest.java              ← business logic tests
│   │   ├── OrderCancellationHandlerTest.java  ← internal handler tests
│   │   └── OrderClientTest.java               ← MockWebServer tests for HTTP clients
│   └── integration/
│       ├── OrderControllerWebMvcTest.java     ← @WebMvcTest
│       ├── OrderRepositoryTest.java           ← @DataJpaTest
│       └── OrderIntegrationTest.java          ← @SpringBootTest, happy path black box
├── payment/
│   ├── unit/
│   │   ├── PaymentServiceTest.java
│   │   └── PaymentGatewayClientTest.java      ← MockWebServer for external gateway
│   └── integration/
│       ├── PaymentControllerWebMvcTest.java
│       ├── PaymentRepositoryTest.java
│       └── PaymentIntegrationTest.java
└── shared/
    └── test/
        └── TestcontainersConfig.java          ← Shared singleton container setup
```

-----

## Layer 1: Unit Tests

A **unit** is a **package** (module), not a single class. Unit tests exercise the real collaborators within a module and mock only at infrastructure boundaries.

### What Gets Mocked

| Boundary              | Mock Strategy       | Example                        |
|-----------------------|---------------------|--------------------------------|
| Repositories          | BDD Mockito         | `given(orderRepository.findById(...))` |
| HTTP clients          | OkHttp MockWebServer| Stand up a local server, verify requests |
| Message producers     | BDD Mockito         | `given(kafkaProducer.send(...))` |
| Other module services | BDD Mockito         | `given(userService.findById(...))` |

### What Does NOT Get Mocked

Everything inside the module under test runs for real: services, mappers, handlers, internal helper classes. If `OrderService` delegates to `OrderCancellationHandler`, both run as real instances in the test.

### Unit Test Example

```java
package com.example.app.order.unit;

import com.example.app.order.dto.OrderCreateRequest;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import com.example.app.order.repository.OrderRepository;
import com.example.app.order.service.OrderService;
import com.example.app.shared.exception.BusinessException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.BDDMockito.then;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(orderRepository);
    }

    @Test
    void shouldCreateOrder_whenRequestIsValid() {
        // Given
        var request = new OrderCreateRequest("john@example.com", new BigDecimal("99.99"));
        given(orderRepository.save(any(OrderEntity.class)))
            .willAnswer(invocation -> invocation.getArgument(0));

        // When
        OrderResponse response = orderService.placeOrder(request);

        // Then
        assertThat(response.customerEmail()).isEqualTo("john@example.com");
        assertThat(response.totalAmount()).isEqualByComparingTo(new BigDecimal("99.99"));
        assertThat(response.status()).isEqualTo("PENDING");
        then(orderRepository).should().save(any(OrderEntity.class));
    }

    @Test
    void shouldThrowBusinessException_whenCancellingShippedOrder() {
        // Given
        var order = new OrderEntity("john@example.com", new BigDecimal("99.99"));
        order.setStatus(OrderStatus.SHIPPED);
        given(orderRepository.findById(1L)).willReturn(Optional.of(order));

        // When / Then
        assertThatThrownBy(() -> orderService.cancelOrder(1L))
            .isInstanceOf(BusinessException.class)
            .hasMessageContaining("shipped");
    }
}
```

### MockWebServer Test Example (HTTP Clients)

MockWebServer tests live in the `unit` package alongside other unit tests. They verify that HTTP client classes correctly handle request construction, response parsing, and error scenarios.

```java
package com.example.app.payment.unit;

import com.example.app.payment.client.PaymentGatewayClient;
import com.example.app.payment.dto.PaymentGatewayResponse;
import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.MockWebServer;
import okhttp3.mockwebserver.RecordedRequest;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PaymentGatewayClientTest {

    private MockWebServer mockWebServer;
    private PaymentGatewayClient client;

    @BeforeEach
    void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
        String baseUrl = mockWebServer.url("/").toString();
        client = new PaymentGatewayClient(baseUrl);
    }

    @AfterEach
    void tearDown() throws IOException {
        mockWebServer.shutdown();
    }

    @Test
    void shouldChargeSuccessfully_whenGatewayReturnsSuccess() throws Exception {
        // Given
        mockWebServer.enqueue(new MockResponse()
            .setResponseCode(200)
            .setHeader("Content-Type", "application/json")
            .setBody("""
                {"transactionId": "txn_123", "status": "CHARGED"}
                """));

        // When
        PaymentGatewayResponse response = client.charge("order_1", new BigDecimal("50.00"));

        // Then
        assertThat(response.transactionId()).isEqualTo("txn_123");
        assertThat(response.status()).isEqualTo("CHARGED");

        RecordedRequest recorded = mockWebServer.takeRequest();
        assertThat(recorded.getMethod()).isEqualTo("POST");
        assertThat(recorded.getPath()).isEqualTo("/charge");
        assertThat(recorded.getBody().readUtf8()).contains("order_1");
    }

    @Test
    void shouldThrowException_whenGatewayReturns500() {
        // Given
        mockWebServer.enqueue(new MockResponse().setResponseCode(500));

        // When / Then
        assertThatThrownBy(() -> client.charge("order_1", new BigDecimal("50.00")))
            .isInstanceOf(PaymentGatewayException.class);
    }
}
```

-----

## Layer 2: Slice Tests

Slice tests load a partial Spring context to verify contracts with the framework.

### @WebMvcTest — HTTP Contract Tests

Tests the controller layer in isolation. The service is mocked with BDD Mockito. These verify request/response serialization, validation, status codes, and error responses.

```java
package com.example.app.order.integration;

import com.example.app.order.controller.OrderController;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.service.OrderService;
import com.example.app.shared.exception.GlobalExceptionHandler;
import com.example.app.shared.exception.ResourceNotFoundException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Optional;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = OrderController.class)
class OrderControllerWebMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    void shouldReturn201_whenOrderCreatedSuccessfully() throws Exception {
        // Given
        var response = new OrderResponse(1L, "john@example.com",
            new BigDecimal("99.99"), "PENDING", LocalDateTime.now(), null);
        given(orderService.placeOrder(any())).willReturn(response);

        // When / Then
        mockMvc.perform(post("/api/v1/order")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerEmail": "john@example.com",
                        "totalAmount": 99.99
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.customerEmail").value("john@example.com"))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    void shouldReturn400_whenEmailIsInvalid() throws Exception {
        // Given — no service setup needed, validation fails before service is called

        // When / Then
        mockMvc.perform(post("/api/v1/order")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerEmail": "not-an-email",
                        "totalAmount": 99.99
                    }
                    """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.customerEmail").exists());
    }

    @Test
    void shouldReturn404_whenOrderNotFound() throws Exception {
        // Given
        given(orderService.findById(999L)).willReturn(Optional.empty());

        // When / Then
        mockMvc.perform(get("/api/v1/order/999"))
            .andExpect(status().isNotFound());
    }
}
```

### @DataJpaTest — Persistence Tests

Tests repository queries and entity mappings against a real database via Testcontainers.

```java
package com.example.app.order.integration;

import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import com.example.app.order.repository.OrderRepository;
import com.example.app.shared.test.TestcontainersConfig;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestcontainersConfig.class)
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldFindOrdersByStatus() {
        // Given
        var pendingOrder = new OrderEntity("alice@example.com", new BigDecimal("50.00"));
        var confirmedOrder = new OrderEntity("bob@example.com", new BigDecimal("75.00"));
        confirmedOrder.setStatus(OrderStatus.CONFIRMED);
        orderRepository.saveAll(List.of(pendingOrder, confirmedOrder));

        // When
        List<OrderEntity> pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

        // Then
        assertThat(pendingOrders).hasSize(1);
        assertThat(pendingOrders.get(0).getCustomerEmail()).isEqualTo("alice@example.com");
    }

    @Test
    void shouldFindOrdersByCustomerEmail() {
        // Given
        orderRepository.save(new OrderEntity("alice@example.com", new BigDecimal("50.00")));
        orderRepository.save(new OrderEntity("alice@example.com", new BigDecimal("75.00")));
        orderRepository.save(new OrderEntity("bob@example.com", new BigDecimal("100.00")));

        // When
        List<OrderEntity> aliceOrders = orderRepository.findByCustomerEmail("alice@example.com");

        // Then
        assertThat(aliceOrders).hasSize(2);
        assertThat(aliceOrders).allMatch(o -> o.getCustomerEmail().equals("alice@example.com"));
    }
}
```

-----

## Layer 3: Full Integration Tests

One test per endpoint (or journey if the entry point is not HTTP). Happy path only. Treat it as a **black box**: send a real HTTP request, let the full stack execute, assert on the response and database state. No mocking — except external HTTP services via MockWebServer, since we don't own them.

### Testcontainers Singleton Configuration

A shared singleton container reused across all test classes. Lives in `shared.test`.

```java
package com.example.app.shared.test;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.PostgreSQLContainer;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

    private static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    static {
        POSTGRES.start();
    }

    @Bean
    @ServiceConnection
    public PostgreSQLContainer<?> postgresContainer() {
        return POSTGRES;
    }
}
```

### Full Integration Test Example

```java
package com.example.app.order.integration;

import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import com.example.app.order.repository.OrderRepository;
import com.example.app.shared.test.TestcontainersConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfig.class)
class OrderIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    void shouldCreateOrderAndPersistToDatabase() throws Exception {
        // When
        mockMvc.perform(post("/api/v1/order")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerEmail": "john@example.com",
                        "totalAmount": 99.99
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.customerEmail").value("john@example.com"))
            .andExpect(jsonPath("$.status").value("PENDING"));

        // Then — verify database state
        List<OrderEntity> orders = orderRepository.findAll();
        assertThat(orders).hasSize(1);
        assertThat(orders.get(0).getCustomerEmail()).isEqualTo("john@example.com");
        assertThat(orders.get(0).getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

### Full Integration Test with External Dependencies

When an endpoint depends on an external HTTP service, the client class is mocked with `@MockitoBean`. The HTTP client's serialization, deserialization, and error handling are already proven by MockWebServer tests in the unit layer — the full integration test's job is to verify the journey from endpoint to database, not to re-test the client.

```java
package com.example.app.payment.integration;

import com.example.app.order.model.OrderEntity;
import com.example.app.order.repository.OrderRepository;
import com.example.app.payment.client.PaymentGatewayClient;
import com.example.app.payment.dto.PaymentGatewayResponse;
import com.example.app.payment.repository.PaymentRepository;
import com.example.app.shared.test.TestcontainersConfig;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfig.class)
class PaymentIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private PaymentRepository paymentRepository;

    @MockitoBean
    private PaymentGatewayClient paymentGatewayClient;

    @BeforeEach
    void setUp() {
        paymentRepository.deleteAll();
    }

    @Test
    void shouldProcessPaymentAndPersistTransaction() throws Exception {
        // Given — an existing order and a successful gateway response
        var order = orderRepository.save(new OrderEntity("john@example.com", new BigDecimal("99.99")));
        given(paymentGatewayClient.charge(any(), any()))
            .willReturn(new PaymentGatewayResponse("txn_abc", "CHARGED"));

        // When
        mockMvc.perform(post("/api/v1/payment")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "orderId": %d,
                        "amount": 99.99
                    }
                    """.formatted(order.getId())))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.transactionId").value("txn_abc"));

        // Then — verify database state
        var payments = paymentRepository.findAll();
        assertThat(payments).hasSize(1);
        assertThat(payments.get(0).getTransactionId()).isEqualTo("txn_abc");
    }
}
```

-----

## Test Libraries and Their Roles

| Library               | Role                                      | Used In             |
|-----------------------|-------------------------------------------|---------------------|
| **JUnit 5**           | Test framework, lifecycle, extensions      | All layers          |
| **BDD Mockito**       | Mocking infrastructure boundaries          | Unit, @WebMvcTest   |
| **@MockitoBean**      | Spring context mock replacement (Spring Boot 3.4+) | @WebMvcTest, Full Integration |
| **AssertJ**           | Fluent assertions                          | All layers          |
| **OkHttp MockWebServer** | Fake HTTP server for client tests       | Unit                |
| **Testcontainers**    | Real database for persistence tests        | @DataJpaTest, Full Integration |
| **MockMvc**           | HTTP testing without a real server          | @WebMvcTest, Full Integration |

-----

## Naming Conventions

### Test Classes

| Test Type              | Pattern                                    | Example                            |
|------------------------|--------------------------------------------|-------------------------------------|
| Unit (service)         | `{Feature}ServiceTest`                     | `OrderServiceTest`                  |
| Unit (handler)         | `{Feature}{Handler}Test`                   | `OrderCancellationHandlerTest`      |
| Unit (HTTP client)     | `{Feature}{Client}Test`                    | `PaymentGatewayClientTest`          |
| WebMvc slice           | `{Feature}ControllerWebMvcTest`            | `OrderControllerWebMvcTest`         |
| DataJpa slice          | `{Feature}RepositoryTest`                  | `OrderRepositoryTest`               |
| Full integration       | `{Feature}IntegrationTest`                 | `OrderIntegrationTest`              |

### Test Methods

All test methods follow the pattern: `should{ExpectedBehavior}_when{Condition}`

```java
void shouldCreateOrder_whenRequestIsValid()
void shouldReturn404_whenOrderNotFound()
void shouldThrowBusinessException_whenCancellingShippedOrder()
void shouldChargeSuccessfully_whenGatewayReturnsSuccess()
void shouldFindOrdersByStatus()  // simple query tests can omit the _when suffix
```

### Test Structure

Every test follows **Given / When / Then** with explicit comments:

```java
@Test
void shouldDoSomething_whenConditionIsMet() {
    // Given
    // ... set up state and mocks

    // When
    // ... execute the action under test

    // Then
    // ... assert outcomes
}
```

For tests where When and Then are a single expression (e.g., asserting an exception), combine them:

```java
@Test
void shouldThrowException_whenInvalidState() {
    // Given
    // ... set up state

    // When / Then
    assertThatThrownBy(() -> service.doSomething())
        .isInstanceOf(BusinessException.class)
        .hasMessageContaining("invalid");
}
```

-----

## Assertion Style

All assertions use **AssertJ**. Never use JUnit's `assertEquals`, `assertTrue`, etc.

### Common Patterns

```java
// Object equality
assertThat(response.customerEmail()).isEqualTo("john@example.com");

// Numeric comparison (BigDecimal)
assertThat(response.totalAmount()).isEqualByComparingTo(new BigDecimal("99.99"));

// Collection assertions
assertThat(orders).hasSize(2);
assertThat(orders).allMatch(o -> o.getStatus() == OrderStatus.PENDING);
assertThat(orders).extracting(OrderResponse::customerEmail)
    .containsExactlyInAnyOrder("alice@example.com", "bob@example.com");

// Exception assertions
assertThatThrownBy(() -> service.cancelOrder(1L))
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("shipped");

// Optional assertions
assertThat(result).isPresent();
assertThat(result).isEmpty();
assertThat(result).hasValueSatisfying(order ->
    assertThat(order.status()).isEqualTo("PENDING")
);

// Null checks
assertThat(response.updatedAt()).isNull();
assertThat(response.createdAt()).isNotNull();
```

-----

## Mocking Style

All mocking uses **BDD Mockito** (`given` / `willReturn` / `then`). Never use classic Mockito (`when` / `thenReturn` / `verify`).

### Common Patterns

```java
// Stubbing
given(orderRepository.findById(1L)).willReturn(Optional.of(order));
given(orderRepository.save(any(OrderEntity.class))).willAnswer(invocation -> invocation.getArgument(0));
given(orderRepository.findAll(any(Pageable.class))).willReturn(Page.empty());

// Verification
then(orderRepository).should().save(any(OrderEntity.class));
then(orderRepository).should(never()).delete(any());
then(kafkaProducer).should().send(argThat(message ->
    message.getKey().equals("order_1")
));
```

-----

## Rules

These rules are non-negotiable. Violating any of them means the test needs refactoring.

1. **Three-layer pyramid.** Unit tests are the bulk. Slice tests verify framework contracts. Full integration tests cover happy-path end-to-end.
2. **A unit is a package.** Unit tests exercise real collaborators within the module. Only infrastructure boundaries are mocked.
3. **Controllers are not unit tested.** Controllers are thin — their behavior is verified by `@WebMvcTest` (slice) and `@SpringBootTest` (full integration).
4. **One full integration test per endpoint/journey.** Happy path only. Black box — assert on response and database state, not internals.
5. **External dependencies in full integration tests are mocked with `@MockitoBean`.** The HTTP client class is replaced in the Spring context. MockWebServer already covers the client's HTTP mechanics in the unit layer — the integration test verifies the journey, not the client.
6. **BDD Mockito only.** Use `given`/`willReturn`/`then`. Never use `when`/`thenReturn`/`verify`.
7. **AssertJ only.** Never use JUnit assertions (`assertEquals`, `assertTrue`, etc.).
8. **OkHttp MockWebServer for HTTP client unit tests only.** MockWebServer lives in the unit layer. Full integration tests mock clients with `@MockitoBean`.
9. **Testcontainers singleton pattern.** One shared container reused across all test classes via `TestcontainersConfig`.
10. **Given / When / Then structure.** Every test has explicit `// Given`, `// When`, `// Then` comments.
11. **Naming: `should{Expected}_when{Condition}`.** Test method names describe behavior, not implementation.
12. **Test classes are package-private.** No `public` modifier on test classes or test methods (JUnit 5 does not require it).
13. **`@BeforeEach` for setup, not constructors.** Test setup goes in `@BeforeEach` methods.
14. **Clean database state.** Full integration tests clean up in `@BeforeEach` (e.g., `repository.deleteAll()`), not `@AfterEach`. Tests must not depend on execution order.
15. **No test inheritance hierarchies.** Avoid abstract base test classes. Prefer composition (e.g., `@Import(TestcontainersConfig.class)`) over inheritance.
16. **Slice tests use `@Import` for Testcontainers.** `@DataJpaTest` classes import `TestcontainersConfig` and disable test database replacement with `@AutoConfigureTestDatabase(replace = Replace.NONE)`.
17. **MockWebServer verifies requests.** Client tests assert both the response parsing and the outgoing request (method, path, headers, body) using `mockWebServer.takeRequest()`.
18. **Never use `@DirtiesContext`.** It forces the Spring context to be destroyed and rebuilt, making tests slow. Design tests so they don't need it — clean database state in `@BeforeEach`, use static MockWebServer with `@BeforeAll`/`@AfterAll`, and avoid mutating shared Spring beans. If a test seems to require `@DirtiesContext`, it is a sign that the test or the production code needs redesigning.
19. **Use `@MockitoBean`, not `@MockBean`.** `@MockBean` is deprecated as of Spring Boot 3.4. Use `org.springframework.test.bean.override.mockito.MockitoBean` for replacing beans in slice tests.
20. **MockWebServer lifecycle in unit tests.** Start in `@BeforeEach`, stop in `@AfterEach`. Each test gets a clean server instance.
