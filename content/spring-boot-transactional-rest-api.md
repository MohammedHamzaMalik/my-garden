---
title: Spring Boot Transactional REST API — From Raw JDBC to Four Layers
date: 2026-02-28
tags:
  - java
  - spring-boot
  - rest-api
  - jpa
  - transactions
  - backend
description: Building a money-transfer REST API with Spring Web, JPA, and @Transactional—the same rollback logic, now behind an HTTP endpoint.
---

# Spring Boot Transactional REST API — From Raw JDBC to Four Layers

After [[database-transactions-and-connection-pooling]], I had the mechanics: HikariCP, `setAutoCommit(false)`, and `rollback()`. The next step was to expose that logic over HTTP so a real client could trigger a transfer—and to let Spring Boot handle the connection and transaction plumbing instead of writing it by hand.

---

## From raw JDBC to Spring Boot

With Spring Boot you don’t write raw SQL or manage the Hikari pool yourself. **Spring Data JPA** talks to the database; **Spring Web** exposes REST endpoints; **HikariCP** is configured from `application.properties`. The same transactional guarantees apply—they’re just wrapped in annotations and layers.

---

## Step 1: Project setup (start.spring.io)

- **Project:** Maven · **Language:** Java · **Spring Boot:** 3.4.x · **Java:** 21  
- **Group:** `com.bank` · **Artifact:** `transaction-api`  
- **Dependencies:** Spring Web, Spring Data JPA, PostgreSQL Driver  

Generate, unzip, open in IntelliJ.

---

## Step 2: Database config

`src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/your_database_name
spring.datasource.username=your_username
spring.datasource.password=your_password

spring.jpa.show-sql=true
```

Spring Boot creates a HikariCP pool (default 10 connections) as soon as it sees the PostgreSQL driver. No manual `HikariConfig` in code.

---

## Step 3: The four layers

All under `src/main/java/com/bank/transactionapi/`.

### 1. Entity — Java ↔ table

`Account` maps to the existing `financial_accounts` table:

```java
@Entity
@Table(name = "financial_accounts")
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "account_name")
    private String accountName;

    private int balance;

    // getters/setters
}
```

### 2. Repository — no SQL strings

Extend `JpaRepository`; Spring generates the SQL. The method name `findByAccountName` becomes a `WHERE account_name = ?` query:

```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    Optional<Account> findByAccountName(String accountName);
}
```

### 3. Service — business logic and `@Transactional`

`TransferService` does the transfer. `@Transactional` means: use one connection, turn off auto-commit for the method, and roll back on any exception. Same idea as the manual JDBC version.

```java
@Service
public class TransferService {

    private final AccountRepository accountRepository;

    public TransferService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void transferMoney(String fromAccount, String toAccount, int amount, boolean simulateCrash) {
        Account sender = accountRepository.findByAccountName(fromAccount)
                .orElseThrow(() -> new RuntimeException("Sender not found"));
        Account receiver = accountRepository.findByAccountName(toAccount)
                .orElseThrow(() -> new RuntimeException("Receiver not found"));

        sender.setBalance(sender.getBalance() - amount);
        accountRepository.save(sender);

        if (simulateCrash) {
            throw new RuntimeException("CRITICAL ERROR! Server crashed mid-transfer!");
        }

        receiver.setBalance(receiver.getBalance() + amount);
        accountRepository.save(receiver);
    }
}
```

### 4. Controller — REST API

`TransferController` exposes a single POST endpoint with query params so we can test success and failure from the command line:

```java
@RestController
@RequestMapping("/api")
public class TransferController {

    private final TransferService transferService;

    public TransferController(TransferService transferService) {
        this.transferService = transferService;
    }

    @PostMapping("/transfer")
    public String executeTransfer(
            @RequestParam String from,
            @RequestParam String to,
            @RequestParam int amount,
            @RequestParam(defaultValue = "false") boolean crash) {

        try {
            transferService.transferMoney(from, to, amount, crash);
            return "Transfer of $" + amount + " successful!";
        } catch (Exception e) {
            return "Transfer failed and rolled back! Reason: " + e.getMessage();
        }
    }
}
```

---

## Step 4: Run and test

Start the app (e.g. run `TransactionApiApplication`), then:

**Test 1 — Successful transfer**

```bash
curl -X POST "http://localhost:8080/api/transfer?from=Account%20A&to=Account%20B&amount=100&crash=false"
```

Response: `Transfer of $100 successful!` — and in the DB, Account A is down 100, Account B is up 100.

**Test 2 — Simulated crash (rollback)**

```bash
curl -X POST "http://localhost:8080/api/transfer?from=Account%20A&to=Account%20B&amount=100&crash=true"
```

Response: `Transfer failed and rolled back! ...` — and in the DB, balances are unchanged. Spring caught the exception, used the same Hikari connection, and rolled back the transaction.

---

## What changed from the plain Java version

| Before (raw JDBC) | After (Spring Boot) |
|-------------------|---------------------|
| Manual `HikariConfig`, `getConnection()` | `application.properties` + auto-configured pool |
| `connection.setAutoCommit(false)` + `try/catch/rollback()` | `@Transactional` on the service method |
| Hand-written `PreparedStatement` and SQL strings | JPA entity + repository method names |
| Single `main()` | REST endpoint; same logic, callable over HTTP |

The guarantees are the same: one transaction, all-or-nothing, rollback on failure. Spring just encodes the pattern so you don’t repeat the boilerplate.

---

## Where this sits in the roadmap

So far: [[dynamic-jvm-plugin-engine]] (loading code), [[java-concurrency-from-race-condition-to-virtual-threads]] (threads and virtual threads), [[database-transactions-and-connection-pooling]] (JDBC + HikariCP + rollback), and now a **transactional REST API** with Spring Boot. Same transfer, same ACID behavior, exposed as an HTTP API—ready to turn into a proper repo with a README and maybe the next feature on my [[java-backend-roadmap]].
