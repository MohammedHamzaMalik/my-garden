---
title: REST API Design — DTOs, Validation, and Structured Error Responses
date: 2026-03-05
tags:
  - java
  - spring-boot
  - rest-api
  - validation
  - backend
description: Move from query params to request/response DTOs, add Bean Validation, and return consistent error payloads with @ControllerAdvice—without losing transactional guarantees.
---

# REST API Design — DTOs, Validation, and Structured Error Responses

After [[backend/why-transactional-isnt-enough-jpa-locking]], the transfer API was correct under concurrency. The next step was to make it **clean for clients**: request/response bodies instead of query params, validation so bad input never reaches the service, and a consistent error shape so frontends and tools can rely on it.

This post tightens the API design while keeping the same transactional and locking guarantees.

---

## Where we left off

The endpoint looked like this:

```http
POST /api/transfer?from=Account%20A&to=Account%20B&amount=100&crash=false
```

Problems:

- Query params are awkward for anything beyond trivial cases (encoding, length, no nesting).
- No structured validation: negative amounts, empty account names, or missing params only fail inside the service or as 500s.
- Errors come back as plain strings; clients can’t reliably parse status or messages.

We’ll fix all three: **DTOs**, **Bean Validation**, and **structured errors**.

---

## 1. Request and response DTOs

### Request DTO

Create a class that describes exactly what the API accepts:

```java
package com.bank.transactionapi.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public class TransferRequest {

    @NotBlank(message = "Sender account name is required")
    private String fromAccount;

    @NotBlank(message = "Receiver account name is required")
    private String toAccount;

    @NotNull(message = "Amount is required")
    @Min(value = 1, message = "Amount must be positive")
    private Integer amount;

    private boolean simulateCrash = false;

    // constructors, getters, setters
}
```

### Response DTO (optional but useful)

For success you can keep a simple message or introduce a small payload:

```java
public record TransferResponse(String message, int amount, String from, String to) {}
```

Or stay with `String` for now; the important part is that **errors** are structured.

---

## 2. Bean Validation dependency

If you didn’t add it at start.spring.io, add the validation starter so `@Valid` and the annotations work:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## 3. Controller: accept body and validate

Switch to `POST` with a JSON body and `@Valid`:

```java
@PostMapping("/transfer")
public ResponseEntity<?> executeTransfer(@Valid @RequestBody TransferRequest request) {
    transferService.transferMoney(
        request.getFromAccount(),
        request.getToAccount(),
        request.getAmount(),
        request.isSimulateCrash()
    );
    return ResponseEntity.ok(
        new TransferResponse(
            "Transfer successful",
            request.getAmount(),
            request.getFromAccount(),
            request.getToAccount()
        )
    );
}
```

If validation fails, Spring won’t call the method; instead we’ll handle it in the advice below and return a structured error.

---

## 4. Structured error response

Define a single shape for API errors so every client gets the same structure:

```java
package com.bank.transactionapi.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.Instant;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_EMPTY)
public record ApiError(
    int status,
    String error,
    String message,
    Instant timestamp,
    String path,
    List<FieldError> fields
) {
    public record FieldError(String field, String message) {}
}
```

Use `status` for the HTTP status code, `error` for the short reason (e.g. `"Conflict"`), `message` for a human-readable description, and `fields` for validation errors.

---

## 5. Global exception handling with `@RestControllerAdvice`

One place to map exceptions to HTTP status and the same `ApiError` shape:

```java
package com.bank.transactionapi.web;

import com.bank.transactionapi.dto.ApiError;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.Instant;
import java.util.List;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        List<ApiError.FieldError> fields = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new ApiError.FieldError(fe.getField(), fe.getDefaultMessage()))
            .collect(Collectors.toList());
        ApiError body = new ApiError(
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Invalid request",
            Instant.now(),
            req.getRequestURI(),
            fields
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(body);
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<ApiError> handleOptimisticLock(OptimisticLockingFailureException ex, HttpServletRequest req) {
        ApiError body = new ApiError(
            HttpStatus.CONFLICT.value(),
            "Conflict",
            "Account balance was updated by another transaction. Please try again.",
            Instant.now(),
            req.getRequestURI(),
            null
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<ApiError> handleIllegalState(IllegalStateException ex, HttpServletRequest req) {
        ApiError body = new ApiError(
            HttpStatus.CONFLICT.value(),
            "Conflict",
            ex.getMessage(),
            Instant.now(),
            req.getRequestURI(),
            null
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleOther(Exception ex, HttpServletRequest req) {
        ApiError body = new ApiError(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal Server Error",
            "Transfer failed. Please try again later.",
            Instant.now(),
            req.getRequestURI(),
            null
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }
}
```

So:

- **Validation errors** → 400 + `fields` populated.
- **Optimistic locking** → 409 with a clear message.
- **Insufficient funds / other business rules** (`IllegalStateException`) → 409.
- **Anything else** → 500 with a generic message (no stack trace in the body).

---

## 6. Example: before vs after

**Before (query params, ad‑hoc errors):**

```http
POST /api/transfer?from=Account%20A&to=Account%20B&amount=-5
→ 200 or 500, plain text body
```

**After (JSON body, validation, structured error):**

```http
POST /api/transfer
Content-Type: application/json

{ "fromAccount": "Account A", "toAccount": "Account B", "amount": -5 }
```

Response (400):

```json
{
  "status": 400,
  "error": "Validation Failed",
  "message": "Invalid request",
  "timestamp": "2026-03-05T12:00:00Z",
  "path": "/api/transfer",
  "fields": [
    { "field": "amount", "message": "Amount must be positive" }
  ]
}
```

Success (200) can return your `TransferResponse` or a simple message; either way, errors are consistent.

---

## 7. What stays the same

- **Transactions:** `@Transactional` on the service is unchanged.
- **Locking:** Optimistic (`@Version`) or pessimistic (`findByAccountNameForUpdate`) is unchanged.
- **Concurrency:** The attack script from [[backend/why-transactional-isnt-enough-jpa-locking]] still applies; only the client now sends a JSON body and receives structured errors.

---

## Where this sits in the roadmap

So far: [[backend/spring-boot-transactional-rest-api]] (first REST API), [[backend/why-transactional-isnt-enough-jpa-locking]] (double spend and JPA locking), and now **API design**—DTOs, validation, and error handling. Next on my [[backend/java-backend-roadmap]]: tests (unit + integration) and then deployment.
