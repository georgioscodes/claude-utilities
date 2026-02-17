---
name: spring-boot-clean-code
description: Use this skill whenever generating, reviewing, or refactoring Java Spring Boot code. Enforces feature-based modular package structure, strict layer separation within modules, opinionated naming conventions, record-based DTOs, manual mappers, centralized exception handling, and inter-module communication rules. Apply to any task involving Spring Boot project scaffolding, new feature development, code review, or architectural guidance.
---

# Spring Boot Clean Code

This skill defines the architecture, conventions, and patterns for writing clean, maintainable Java Spring Boot applications. It enforces a **feature-based modular package structure** with strict layer separation within each module.

---

## Core Principle: Feature-Based Modules

Code is organized by **business domain/feature**, not by technical layer. Each feature is a self-contained module with its own internal layers.

### Correct Structure (feature-based)

```
com.example.app
├── order/
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── model/
│   ├── dto/
│   └── mapper/
├── payment/
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── model/
│   ├── dto/
│   └── mapper/
├── user/
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── model/
│   ├── dto/
│   └── mapper/
└── shared/
    ├── config/
    ├── security/
    ├── exception/
    └── util/
```

### Wrong Structure (layer-based) — NEVER use this

```
com.example.app
├── controller/
│   ├── OrderController.java
│   ├── PaymentController.java
│   └── UserController.java
├── service/
│   ├── OrderService.java
│   ├── PaymentService.java
│   └── UserService.java
├── repository/
│   ├── OrderRepository.java
│   ...
```

---

## Module Layers

Every feature module contains exactly these six sub-packages. Do not add extra sub-packages. Do not omit any unless the module genuinely does not need that layer (e.g., a module with no REST endpoints has no `controller` package).

### Layer Responsibilities

| Layer | Package | Responsibility |
|-------|---------|---------------|
| **Controller** | `feature.controller` | HTTP entry point. Receives requests, delegates to service, returns responses. No business logic. |
| **Service** | `feature.service` | Business logic only. Orchestrates operations, enforces business rules. |
| **Repository** | `feature.repository` | Data access. Spring Data interfaces or custom query implementations. |
| **Model** | `feature.model` | JPA entities. Internal to the module — never exposed outside. |
| **DTO** | `feature.dto` | Record-based data transfer objects. The public contract of the module. |
| **Mapper** | `feature.mapper` | Manual conversion between entities and DTOs. Static methods, no dependencies. |

### Layer Dependency Rules

```
controller → service → repository → model
     ↓           ↓
    dto         dto
     ↑           ↑
   mapper      mapper
```

- Controllers depend on services and DTOs.
- Services depend on repositories, DTOs, and mappers.
- Repositories depend on model/entities.
- Mappers depend on both model and DTOs.
- **Model/entities NEVER leak outside the module.** DTOs are the only public contract.

---

## Naming Conventions

These are strict. Follow them exactly.

### Classes

| Layer | Pattern | Example |
|-------|---------|---------|
| Controller | `{Feature}Controller` | `OrderController` |
| Service (default) | `{Feature}Service` | `OrderService` |
| Service Interface (only when needed) | `{Feature}Service` | `OrderService` (interface) |
| Service Implementation (only when interface exists) | `{Feature}ServiceImpl` | `OrderServiceImpl` |
| Repository | `{Feature}Repository` | `OrderRepository` |
| Entity | `{Feature}Entity` | `OrderEntity` |
| Request DTO | `{Feature}{Action}Request` | `OrderCreateRequest` |
| Response DTO | `{Feature}Response` | `OrderResponse` |
| Mapper | `{Feature}Mapper` | `OrderMapper` |

### Methods

| Layer | Convention | Examples |
|-------|-----------|----------|
| Controller | HTTP-verb aligned | `create()`, `getById()`, `update()`, `delete()`, `getAll()` |

### REST API Paths

API paths use **singular** resource names. Never use plurals.

| Correct | Wrong |
|---------|-------|
| `/api/v1/order` | `/api/v1/orders` |
| `/api/v1/payment` | `/api/v1/payments` |
| `/api/v1/user` | `/api/v1/users` |
| Service | Business-action aligned | `placeOrder()`, `cancelOrder()`, `findById()`, `findAll()` |
| Repository | Spring Data conventions | `findByEmail()`, `findByStatusAndCreatedAtAfter()` |
| Mapper | `toDto()`, `toEntity()`, `toDtoList()` | Static methods only |

### Packages

- Always lowercase, singular nouns: `order`, `payment`, `user` — never `orders`, `payments`, `users`.
- Sub-packages follow layer names exactly: `controller`, `service`, `repository`, `model`, `dto`, `mapper`.

---

## Code Examples

### Entity (model layer)

```java
package com.example.app.order.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
public class OrderEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String customerEmail;

    @Column(nullable = false)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    protected OrderEntity() {
        // JPA requires a no-arg constructor
    }

    public OrderEntity(String customerEmail, BigDecimal totalAmount) {
        this.customerEmail = customerEmail;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }

    // Getters and setters — no Lombok on entities.
    // Only expose setters for fields that are legitimately mutable.

    public Long getId() {
        return id;
    }

    public String getCustomerEmail() {
        return customerEmail;
    }

    public BigDecimal getTotalAmount() {
        return totalAmount;
    }

    public OrderStatus getStatus() {
        return status;
    }

    public void setStatus(OrderStatus status) {
        this.status = status;
        this.updatedAt = LocalDateTime.now();
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public LocalDateTime getUpdatedAt() {
        return updatedAt;
    }
}
```

```java
package com.example.app.order.model;

public enum OrderStatus {
    PENDING,
    CONFIRMED,
    SHIPPED,
    DELIVERED,
    CANCELLED
}
```

### DTOs (record-based)

```java
package com.example.app.order.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import java.math.BigDecimal;

public record OrderCreateRequest(

    @NotBlank(message = "Customer email is required")
    @Email(message = "Must be a valid email address")
    String customerEmail,

    @NotNull(message = "Total amount is required")
    @Positive(message = "Total amount must be positive")
    BigDecimal totalAmount
) {}
```

```java
package com.example.app.order.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record OrderResponse(
    Long id,
    String customerEmail,
    BigDecimal totalAmount,
    String status,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

### Mapper (manual, static methods)

```java
package com.example.app.order.mapper;

import com.example.app.order.dto.OrderCreateRequest;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.model.OrderEntity;

import java.util.List;

public final class OrderMapper {

    private OrderMapper() {
        // Utility class — no instantiation
    }

    public static OrderEntity toEntity(OrderCreateRequest request) {
        return new OrderEntity(request.customerEmail(), request.totalAmount());
    }

    public static OrderResponse toDto(OrderEntity order) {
        return new OrderResponse(
            order.getId(),
            order.getCustomerEmail(),
            order.getTotalAmount(),
            order.getStatus().name(),
            order.getCreatedAt(),
            order.getUpdatedAt()
        );
    }

    public static List<OrderResponse> toDtoList(List<OrderEntity> orders) {
        return orders.stream()
            .map(OrderMapper::toDto)
            .toList();
    }
}
```

### Repository

```java
package com.example.app.order.repository;

import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface OrderRepository extends JpaRepository<OrderEntity, Long> {

    List<OrderEntity> findByCustomerEmail(String customerEmail);

    List<OrderEntity> findByStatus(OrderStatus status);
}
```

### Service

```java
package com.example.app.order.service;

import com.example.app.order.dto.OrderCreateRequest;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.mapper.OrderMapper;
import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import com.example.app.order.repository.OrderRepository;
import com.example.app.shared.dto.PagedResponse;
import com.example.app.shared.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderResponse placeOrder(OrderCreateRequest request) {
        OrderEntity order = OrderMapper.toEntity(request);
        OrderEntity saved = orderRepository.save(order);
        return OrderMapper.toDto(saved);
    }

    @Transactional(readOnly = true)
    public Optional<OrderResponse> findById(Long id) {
        return orderRepository.findById(id)
            .map(OrderMapper::toDto);
    }

    @Transactional(readOnly = true)
    public PagedResponse<OrderResponse> findAll(Pageable pageable) {
        Page<OrderEntity> page = orderRepository.findAll(pageable);
        Page<OrderResponse> mapped = page.map(OrderMapper::toDto);
        return PagedResponse.from(mapped);
    }

    public OrderResponse cancelOrder(Long id) {
        OrderEntity order = orderRepository.findById(id)
            .orElseThrow(() -> new BusinessException("Order not found with id: " + id));

        if (order.getStatus() == OrderStatus.SHIPPED || order.getStatus() == OrderStatus.DELIVERED) {
            throw new BusinessException("Cannot cancel an order that has already been " + order.getStatus().name().toLowerCase());
        }

        order.setStatus(OrderStatus.CANCELLED);
        OrderEntity saved = orderRepository.save(order);
        return OrderMapper.toDto(saved);
    }
}
```

> **When to introduce an interface:** Only create a service interface when there is a genuine need — multiple implementations, a clear need for mocking boundaries beyond what frameworks like Mockito already handle, or a published API contract for other teams. Do not create an interface "just in case". A single class with `@Service` is the default.
```

### Controller

```java
package com.example.app.order.controller;

import com.example.app.order.dto.OrderCreateRequest;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.service.OrderService;
import com.example.app.shared.dto.PagedResponse;
import com.example.app.shared.exception.ResourceNotFoundException;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/order")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderCreateRequest request) {
        OrderResponse response = orderService.placeOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
    }

    @GetMapping
    public ResponseEntity<PagedResponse<OrderResponse>> getAll(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
        return ResponseEntity.ok(orderService.findAll(pageable));
    }

    @PatchMapping("/{id}/cancel")
    public ResponseEntity<OrderResponse> cancel(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.cancelOrder(id));
    }
}
```

---

## Single Responsibility Principle

Classes and methods must each have one clear reason to exist. When a service grows too large, split it into focused classes within the same module.

### When to Split a Service

If you find yourself mentally grouping a service's methods into distinct areas of concern, it's time to split. The original service becomes a thin orchestrator that delegates to focused internal classes.

### Example: Splitting OrderService

The `OrderService` handles creation, querying, and cancellation. If the order module grows to include fulfillment, refunds, and status tracking, the service becomes too large. Split it:

```
com.example.app.order
├── controller/
│   └── OrderController.java
├── service/
│   ├── OrderService.java              ← public, the module's entry point
│   ├── OrderCancellationHandler.java  ← package-private
│   └── OrderFulfillmentHandler.java   ← package-private
├── repository/
├── model/
├── dto/
└── mapper/
```

### Package-Private Internal Classes

Helper classes that exist only to decompose a service within the module must be **package-private** (no `public` modifier). This prevents other modules from depending on internal implementation details.

```java
package com.example.app.order.service;

import com.example.app.order.model.OrderEntity;
import com.example.app.order.model.OrderStatus;
import com.example.app.order.repository.OrderRepository;
import com.example.app.shared.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
class OrderCancellationHandler {

    private final OrderRepository orderRepository;

    OrderEntity cancel(Long id) {
        OrderEntity order = orderRepository.findById(id)
            .orElseThrow(() -> new BusinessException("Order not found with id: " + id));

        if (order.getStatus() == OrderStatus.SHIPPED || order.getStatus() == OrderStatus.DELIVERED) {
            throw new BusinessException("Cannot cancel an order that has already been " + order.getStatus().name().toLowerCase());
        }

        order.setStatus(OrderStatus.CANCELLED);
        return orderRepository.save(order);
    }
}
```

The public `OrderService` delegates to it:

```java
package com.example.app.order.service;

import com.example.app.order.dto.OrderCreateRequest;
import com.example.app.order.dto.OrderResponse;
import com.example.app.order.mapper.OrderMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderCancellationHandler cancellationHandler;

    public OrderResponse placeOrder(OrderCreateRequest request) {
        // ...
    }

    public OrderResponse cancelOrder(Long id) {
        return OrderMapper.toDto(cancellationHandler.cancel(id));
    }

    // ...
}
```

### Method-Level SRP

Methods should do one thing. If a method has multiple logical steps, extract each step into a well-named private method.

```java
// BAD — method does too many things
public OrderResponse placeOrder(OrderCreateRequest request) {
    // 15 lines of validation
    // 10 lines of discount calculation
    // 10 lines of inventory check
    // 5 lines of persistence
}

// GOOD — each step is a focused method
public OrderResponse placeOrder(OrderCreateRequest request) {
    OrderEntity order = OrderMapper.toEntity(request);
    applyDiscounts(order);
    reserveInventory(order);
    OrderEntity saved = orderRepository.save(order);
    return OrderMapper.toDto(saved);
}
```

### Rules

- **Split by responsibility, not by size alone.** A small class doing two things is worse than a medium class doing one thing.
- **Internal helper classes are package-private.** Only the main service class is `public`.
- **Other modules never see the split.** From the outside, they only know `OrderService`. The internal decomposition is invisible.
- **Methods do one thing.** If you need comments to separate sections of a method, extract those sections into named methods.

---

Modules MUST communicate through services only. Direct access to another module's repository, model, or mapper is **strictly forbidden**.

### Correct: Module A calls Module B's service

```java
package com.example.app.payment.service;

import com.example.app.order.dto.OrderResponse;
import com.example.app.order.service.OrderService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class PaymentService {

    private final OrderService orderService;

    public PaymentResponse processPayment(Long orderId, PaymentRequest request) {
        OrderResponse order = orderService.findById(orderId)
            .orElseThrow(() -> new BusinessException("Order not found"));

        // Payment business logic using order DTO data — never the entity.
        // ...
    }
}
```

### Forbidden: Module A imports Module B's repository or entity

```java
// NEVER DO THIS
import com.example.app.order.repository.OrderRepository;  // FORBIDDEN
import com.example.app.order.model.Order;                  // FORBIDDEN
```

### What can be imported across modules

| Importable | Example |
|-----------|---------|
| Service interface | `com.example.app.order.service.OrderService` |
| DTOs | `com.example.app.order.dto.OrderResponse` |
| Shared exceptions | `com.example.app.shared.exception.BusinessException` |

| NOT importable | Why |
|---------------|-----|
| Repository | Internal data access detail |
| Entity/Model | Internal persistence detail |
| Mapper | Internal conversion detail |
| Service implementation | Depend on interface, not implementation |

---

## Shared Package

The `shared` package holds cross-cutting concerns that are not specific to any one feature.

```
com.example.app.shared
├── config/          # App-wide configuration (@Configuration beans)
├── dto/             # Reusable DTOs used across modules (e.g., PagedResponse)
├── security/        # Security filters, authentication config
├── exception/       # Global exception classes and handler
└── util/            # Common utility classes (use sparingly)
```

### Rules for `shared`

- Only put things here if they are used by **two or more modules**.
- If something is only used by one module, it belongs in that module.
- `shared` has no service, repository, or controller packages — it is infrastructure, not a feature.

---

## Validation Strategy

Validation is split into two distinct layers:

### 1. Input Validation — on DTOs at entry points

Use Bean Validation annotations on request DTOs. Controllers enforce with `@Valid`.

```java
public record OrderCreateRequest(
    @NotBlank(message = "Customer email is required")
    @Email(message = "Must be a valid email address")
    String customerEmail,

    @NotNull(message = "Total amount is required")
    @Positive(message = "Total amount must be positive")
    BigDecimal totalAmount
) {}
```

```java
@PostMapping
public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderCreateRequest request) {
    // If validation fails, MethodArgumentNotValidException is thrown
    // and caught by the global exception handler. Controller stays clean.
    return ResponseEntity.status(HttpStatus.CREATED).body(orderService.placeOrder(request));
}
```

Input validation answers: **"Is this data well-formed?"** (Is the email valid? Is the amount positive? Is the name non-blank?)

### 2. Business Validation — in the service layer

Business rules live in services. When violated, throw module-specific or shared exceptions.

```java
public OrderResponse cancelOrder(Long id) {
    OrderEntity order = orderRepository.findById(id)
        .orElseThrow(() -> new BusinessException("Order not found with id: " + id));

    if (order.getStatus() == OrderStatus.SHIPPED) {
        throw new BusinessException("Cannot cancel a shipped order");
    }

    order.setStatus(OrderStatus.CANCELLED);
    return OrderMapper.toDto(orderRepository.save(order));
}
```

Business validation answers: **"Is this operation allowed given the current state?"** (Can this order be cancelled? Does the user have sufficient balance? Is this coupon still valid?)

### 3. Null Safety — Optional for nullable returns

Public service methods that might not find a result return `Optional<T>`.

```java
// In the service
Optional<OrderResponse> findById(Long id);

// The caller decides how to handle absence
orderService.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Order", id));
```

---

## Pagination

Collection endpoints must return paginated responses. Never return an unbounded `List` from a controller.

### Paginated Response DTO

Define a generic, reusable page wrapper in the `shared` package since it is used across all modules.

```java
package com.example.app.shared.dto;

import java.util.List;

public record PagedResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PagedResponse<T> from(org.springframework.data.domain.Page<T> page) {
        return new PagedResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isLast()
        );
    }
}
```

### Controller

The controller receives pagination parameters via Spring's `Pageable` and returns a `PagedResponse`. Use `@PageableDefault` to set sensible defaults.

```java
@GetMapping
public ResponseEntity<PagedResponse<OrderResponse>> getAll(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
    return ResponseEntity.ok(orderService.findAll(pageable));
}
```

### Service

The service accepts `Pageable`, delegates to the repository, and maps the result.

```java
@Transactional(readOnly = true)
public PagedResponse<OrderResponse> findAll(Pageable pageable) {
    Page<OrderEntity> page = orderRepository.findAll(pageable);
    Page<OrderResponse> mapped = page.map(OrderMapper::toDto);
    return PagedResponse.from(mapped);
}
```

### Repository

No changes needed — Spring Data's `JpaRepository` already supports `Page<T> findAll(Pageable pageable)` out of the box.

### Rules

- **All collection endpoints are paginated.** No controller method returns a raw `List` to the client.
- **Default page size is set at the controller.** Use `@PageableDefault` — do not rely on Spring's global default.
- **Sorting is declared at the controller.** The default sort field and direction are explicit in `@PageableDefault`.
- **`PagedResponse` lives in `shared.dto`.** It is a generic, reusable wrapper — not module-specific.
- **Services return `PagedResponse`, not `Page`.** The Spring `Page` type stays internal — the controller and external consumers see the record-based DTO.

---

All exception handling goes through a single `@RestControllerAdvice` in the `shared.exception` package.

### Exception Classes

```java
package com.example.app.shared.exception;

public class BusinessException extends RuntimeException {

    public BusinessException(String message) {
        super(message);
    }
}
```

```java
package com.example.app.shared.exception;

public class ResourceNotFoundException extends RuntimeException {

    private final String resourceName;
    private final Object resourceId;

    public ResourceNotFoundException(String resourceName, Object resourceId) {
        super(String.format("%s not found with id: %s", resourceName, resourceId));
        this.resourceName = resourceName;
        this.resourceId = resourceId;
    }

    public String getResourceName() {
        return resourceName;
    }

    public Object getResourceId() {
        return resourceId;
    }
}
```

### Error Response DTO

```java
package com.example.app.shared.exception;

import java.time.LocalDateTime;
import java.util.Map;

public record ErrorResponse(
    int status,
    String message,
    LocalDateTime timestamp,
    Map<String, String> errors
) {
    public ErrorResponse(int status, String message) {
        this(status, message, LocalDateTime.now(), Map.of());
    }
}
```

### Global Exception Handler

```java
package com.example.app.shared.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        ErrorResponse error = new ErrorResponse(HttpStatus.BAD_REQUEST.value(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(fe -> fieldErrors.put(fe.getField(), fe.getDefaultMessage()));

        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            java.time.LocalDateTime.now(),
            fieldErrors
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred"
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

---

## Logging Strategy

Logging follows a **boundary-first** approach. Requests, responses, HTTP client calls, and database operations are logged automatically by infrastructure — never manually in application code. Business logic logging is the exception, not the norm.

### Structured JSON Logging

All environments use structured JSON logging via Logback with the Logstash encoder. This ensures logs are machine-parseable and compatible with centralized log aggregation tools.

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

### Correlation ID via MDC

Every incoming request gets a correlation ID propagated through MDC. This allows tracing a single request across all log entries it produces. The correlation ID is set by a filter in `shared.config` and cleared after the request completes.

```java
package com.example.app.shared.config;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-Id";
    private static final String CORRELATION_ID_MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        try {
            String correlationId = request.getHeader(CORRELATION_ID_HEADER);
            if (correlationId == null || correlationId.isBlank()) {
                correlationId = UUID.randomUUID().toString();
            }
            MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
            response.setHeader(CORRELATION_ID_HEADER, correlationId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY);
        }
    }
}
```

### What Gets Logged and Where

| Concern | Logged By | Manual Log Statements? |
|---------|-----------|----------------------|
| Inbound HTTP requests/responses | Interceptor or filter in `shared` | **No** — never log manually in controllers |
| Outbound HTTP client calls/responses | Client interceptor (`ClientHttpRequestInterceptor` or `ExchangeFilterFunction`) | **No** — never log manually around client calls |
| Database operations | Infrastructure-level (Hibernate/JPA logging config) | **No** — never log manually in repositories or around DB calls |
| Complex business logic decisions | Manual `log.debug()` in the service | **Yes** — only when a non-obvious decision path needs traceability |
| Errors and exceptions | `GlobalExceptionHandler` + stack traces | **Yes** — always log at `ERROR` level with full stack trace |

### Log Levels

| Level | Use For |
|-------|---------|
| `ERROR` | Exceptions, unexpected failures, anything requiring attention |
| `WARN` | Recoverable issues, degraded behavior, fallback paths taken |
| `INFO` | Boundary logs (requests, responses, client calls) — handled by interceptors |
| `DEBUG` | Complex business logic trace points — the only manual logs in services |

### When to Add a Manual Log Statement

Only add an explicit log statement in a service when **all** of these are true:

1. A complex calculation or decision is being made (not simple CRUD).
2. The outcome would be difficult to reconstruct from boundary logs alone.
3. The log message explains **why** a path was taken, not **what** happened.

```java
// GOOD — explains a non-obvious decision
log.debug("Applying tiered discount: tier={}, rate={}, because customerOrderCount={}", tier, rate, orderCount);

// BAD — restates what the code already does
log.info("Saving order to database");

// BAD — duplicates what interceptors already log
log.info("Received request to create order for customer: {}", request.customerEmail());
```

### Rules

- **No manual logging of HTTP requests, responses, or client calls.** Interceptors handle this.
- **No manual logging of database operations.** Infrastructure handles this.
- **No logging in controllers.** Controllers are thin delegation layers — nothing to log.
- **No logging in mappers or DTOs.** These are pure data transformation — nothing to log.
- **Errors always logged with stack traces.** The `GlobalExceptionHandler` handles this centrally.
- **Correlation ID present on every log entry.** The MDC filter ensures this automatically.
- **Structured JSON format in all environments.** Logback + Logstash encoder, no plain-text patterns.

---



These rules are non-negotiable. Violating any of them means the code needs refactoring.

1. **Feature-based packages.** Top-level packages are business domains, not layers.
2. **Six layers per module.** `controller`, `service`, `repository`, `model`, `dto`, `mapper`.
3. **Entities are suffixed with `Entity`.** `OrderEntity`, `PaymentEntity`, `UserEntity`.
4. **Entities never leave their module.** DTOs are the only public contract.
5. **Inter-module communication through services only.** Never import another module's repository, entity, or mapper.
6. **Record-based DTOs.** All DTOs are Java records. No Lombok on DTOs, no mutable classes.
7. **Manual mappers with static methods.** Mapper classes are `final` with a private constructor. All methods are `static`.
8. **Input validation on DTOs.** Bean Validation annotations on request records. Controllers enforce with `@Valid`.
9. **Business validation in services.** Services throw exceptions for business rule violations.
10. **Optional for nullable returns.** Service methods that might return nothing use `Optional<T>`.
11. **Lombok `@RequiredArgsConstructor` for dependency injection.** All `final` fields injected via Lombok-generated constructor. No `@Autowired`, no hand-written constructors for DI.
12. **No unnecessary interfaces.** Services are concrete classes by default. Only introduce an interface when there is a genuine need (multiple implementations, published API contract).
13. **Single Responsibility Principle.** Classes and methods do one thing. Large services are split into package-private helper classes within the same module.
14. **Package-private by default for internals.** Helper classes, internal handlers, and anything not needed by other modules must be package-private (no `public` modifier).
15. **Centralized exception handling.** One `@RestControllerAdvice` in `shared.exception`. No try-catch in controllers.
16. **Naming follows conventions exactly.** See the naming conventions table — no deviations.
17. **Shared package is for cross-cutting only.** If it's used by one module, it belongs in that module.
18. **No manual logging of HTTP or database operations.** Interceptors and infrastructure handle boundary logging. Never add log statements in controllers, around client calls, or around repository calls.
19. **Structured JSON logging.** Logback + Logstash encoder in all environments.
20. **Correlation ID on every request.** MDC filter propagates a correlation ID across the entire request lifecycle.
21. **Business logic logging is the exception.** Only log manually in services when a complex decision path needs traceability. Use `DEBUG` level.
22. **All collection endpoints are paginated.** Controllers never return a raw `List`. Use `PagedResponse` with `Pageable` and `@PageableDefault`.
