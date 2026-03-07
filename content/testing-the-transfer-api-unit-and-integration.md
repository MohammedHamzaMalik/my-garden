---
title: Testing the Transfer API — Unit and Integration Tests
date: 2026-03-06
tags:
  - java
  - spring-boot
  - testing
  - junit
  - mockito
  - backend
description: Unit tests for TransferService with mocked repositories, and integration tests for the REST endpoint with MockMvc—validation, success, and error paths.
---

# Testing the Transfer API — Unit and Integration Tests

After [[rest-api-design-dtos-validation-error-handling]], the transfer API had DTOs, validation, and structured errors. The next step was to **lock that behavior in with tests**: unit tests for the service (mocked repository) and integration tests for the HTTP layer so refactors and new features don’t break what already works.

This post covers a practical testing setup for the same transfer API, without touching the database in unit tests and with a real (or test) DB only where we need it.

---

## What we’re testing

- **TransferService** — business logic: load accounts, check balance, deduct/credit, handle “simulate crash.” We test this in isolation with a **mocked** `AccountRepository`.
- **REST endpoint** — HTTP contract: request body → validation → status and response/error body. We test this with **MockMvc** (or full integration with TestRestTemplate) so we hit the real controller, validation, and exception advice.

---

## 1. Dependencies

Spring Boot already brings JUnit 5 and Mockito. For MockMvc (and optional AssertJ) you typically have:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

That’s enough for the examples below. Use `@SpringBootTest` only when you want a full context (e.g. real DB or test containers); for “slice” tests we’ll use `@WebMvcTest` and `@ExtendWith(MockitoExtension.class)`.

---

## 2. Unit tests for `TransferService`

Goal: verify that with **given** account data from the repository, the service **does the right math** and **throws** when it should (e.g. insufficient funds, missing account).

Example (adjust package and class names to match yours):

```java
package com.bank.transactionapi.service;

import com.bank.transactionapi.Account;
import com.bank.transactionapi.AccountRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class TransferServiceTest {

    @Mock
    private AccountRepository accountRepository;

    @InjectMocks
    private TransferService transferService;

    @Test
    void transferMoney_deductsFromSenderAndCreditsReceiver() {
        Account sender = new Account();
        sender.setId(1L);
        sender.setAccountName("Account A");
        sender.setBalance(200);

        Account receiver = new Account();
        receiver.setId(2L);
        receiver.setAccountName("Account B");
        receiver.setBalance(50);

        when(accountRepository.findByAccountNameForUpdate("Account A")).thenReturn(Optional.of(sender));
        when(accountRepository.findByAccountNameForUpdate("Account B")).thenReturn(Optional.of(receiver));

        transferService.transferMoney("Account A", "Account B", 100, false);

        assert sender.getBalance() == 100;
        assert receiver.getBalance() == 150;
        verify(accountRepository).save(sender);
        verify(accountRepository).save(receiver);
    }

    @Test
    void transferMoney_throwsWhenInsufficientFunds() {
        Account sender = new Account();
        sender.setAccountName("Account A");
        sender.setBalance(50);

        when(accountRepository.findByAccountNameForUpdate("Account A")).thenReturn(Optional.of(sender));
        when(accountRepository.findByAccountNameForUpdate("Account B")).thenReturn(Optional.of(new Account()));

        assertThatThrownBy(() ->
            transferService.transferMoney("Account A", "Account B", 100, false))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Insufficient funds");
    }

    @Test
    void transferMoney_throwsWhenSenderNotFound() {
        when(accountRepository.findByAccountNameForUpdate("Account A")).thenReturn(Optional.empty());

        assertThatThrownBy(() ->
            transferService.transferMoney("Account A", "Account B", 100, false))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("Sender not found");
    }
}
```

Takeaways:

- **No Spring context** — plain JUnit + Mockito; fast and focused on service logic.
- **Repository is mocked** — we control exactly what “DB” returns and we verify `save` calls.
- You can add a test that calls with `simulateCrash = true` and asserts that an exception is thrown and that the receiver is *not* credited (if your service leaves receiver unchanged in that path).

---

## 3. Integration tests for the REST endpoint

Goal: hit the real controller (and optionally validation + `@RestControllerAdvice`) and assert on HTTP status and response body. Two common styles:

### Option A: `@WebMvcTest` (controller slice, mocked service)

Only the web layer is loaded; you mock `TransferService` so no real DB is needed.

```java
package com.bank.transactionapi.web;

import com.bank.transactionapi.TransferController;
import com.bank.transactionapi.service.TransferService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.verify;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(TransferController.class)
class TransferControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private TransferService transferService;

    @Test
    void transfer_validRequest_returns200() throws Exception {
        String body = """
            {"fromAccount":"Account A","toAccount":"Account B","amount":100,"simulateCrash":false}
            """;

        mockMvc.perform(post("/api/transfer")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.message").exists());

        verify(transferService).transferMoney("Account A", "Account B", 100, false);
    }

    @Test
    void transfer_negativeAmount_returns400WithFieldErrors() throws Exception {
        String body = """
            {"fromAccount":"Account A","toAccount":"Account B","amount":-10,"simulateCrash":false}
            """;

        mockMvc.perform(post("/api/transfer")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.status").value(400))
            .andExpect(jsonPath("$.fields[?(@.field == 'amount')]").exists());
    }

    @Test
    void transfer_serviceThrowsIllegalState_returns409() throws Exception {
        String body = """
            {"fromAccount":"Account A","toAccount":"Account B","amount":1000,"simulateCrash":false}
            """;
        doThrow(new IllegalStateException("Insufficient funds")).when(transferService)
            .transferMoney(any(), any(), anyInt(), anyBoolean());

        mockMvc.perform(post("/api/transfer")
                .contentType(MediaType.APPLICATION_JSON)
                .content(body))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.message").value("Insufficient funds"));
    }
}
```

Here we’re testing that:

- Valid JSON → 200 and service is called with the right arguments.
- Invalid input (e.g. negative amount) → 400 and structured error with `fields`.
- Service throwing `IllegalStateException` → 409 and our `ApiError` message.

### Option B: `@SpringBootTest` + TestRestTemplate (full app, real or test DB)

If you want to hit the real service and DB (or H2/testcontainers), use:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class TransferApiIntegrationTest {

    @LocalServerPort
    int port;

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void transfer_e2e_success() {
        String body = "{\"fromAccount\":\"Account A\",\"toAccount\":\"Account B\",\"amount\":10,\"simulateCrash\":false}";
        ResponseEntity<String> res = restTemplate.postForEntity(
            "http://localhost:" + port + "/api/transfer",
            new HttpEntity<>(body, headers("application/json")),
            String.class
        );
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

You’d seed the DB in `@BeforeEach` or with a script and then assert on both HTTP response and final balances if needed.

---

## 4. What to add over time

- **More validation cases** — blank strings, missing `fromAccount`/`toAccount`, etc., and assert on `status` and `fields`.
- **Optimistic locking** — in an integration test, trigger `OptimisticLockingFailureException` (e.g. via two concurrent requests or a test double) and assert 409 and the “updated by another transaction” message.
- **Repository tests** — if you add custom queries (e.g. `findByAccountNameForUpdate`), use `@DataJpaTest` and an in-memory or test DB to verify the generated SQL and locking behavior.

---

## Where this sits in the roadmap

So far: [[spring-boot-transactional-rest-api]], [[why-transactional-isnt-enough-jpa-locking]], [[rest-api-design-dtos-validation-error-handling]], and now **tests**—unit for the service, integration for the API and error handling. Next on my [[java-backend-roadmap]]: deployment (e.g. container or cloud) and then iterating on features with the same discipline.
