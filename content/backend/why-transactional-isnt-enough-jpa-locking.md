---
title: Why @Transactional Isn’t Enough — Solving the Double Spend with JPA Locking
date: 2026-03-05
tags:
  - java
  - spring-boot
  - jpa
  - transactions
  - concurrency
  - postgresql
description: Reproduce the “double spend” (lost update) in a transactional Spring Boot API, then fix it with optimistic locking (@Version) and pessimistic locking (SELECT FOR UPDATE).
---

# Why `@Transactional` Isn’t Enough — Solving the Double Spend with JPA Locking

In [[backend/spring-boot-transactional-rest-api]] I built a money transfer API with Spring Web + Spring Data JPA + `@Transactional`. It rolls back on failures and keeps “all-or-nothing” guarantees.

But there’s a **bigger** reliability problem that `@Transactional` alone does *not* solve:

## The double spend (lost update) problem

If two requests hit `/api/transfer` at the same time, both can read the same balance and both can write an updated balance based on stale data.

Example:

- Account A has **$100**
- Two requests concurrently transfer **$100** from A → B
- Both transactions read `balance = 100`
- Both compute `100 - 100 = 0`
- Both persist `0`

Outcome: you let **$200** leave a **$100** account.

This is a classic database concurrency anomaly: **Lost Update**.

This post is my “break it, then fix it” deep dive. We’ll:

1. Prove the bug with a concurrent attack script.
2. Fix it with **Optimistic Locking** (`@Version`).
3. Fix it with **Pessimistic Locking** (`SELECT … FOR UPDATE` via `@Lock`).

---

## Block 1 — The attack (simulate concurrent requests)

### Step 1: Seed balances for the test

Set Account A to **100** and Account B to **0** (or any known baseline):

```sql
UPDATE financial_accounts SET balance = 100 WHERE account_name = 'Account A';
UPDATE financial_accounts SET balance = 0 WHERE account_name = 'Account B';
```

Also, make sure your service has a basic “insufficient funds” guard (this matters later):

```java
if (sender.getBalance() < amount) {
  throw new IllegalStateException("Insufficient funds");
}
```

### Step 2: Create a Java “attack script”

This is a separate tiny Java program (not inside the Spring Boot app) that fires **10 concurrent HTTP POST requests**.

Create a file like `AttackDoubleSpend.java`:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class AttackDoubleSpend {
  public static void main(String[] args) throws Exception {
    int attackers = 10;
    String url =
      "http://localhost:8080/api/transfer?from=Account%20A&to=Account%20B&amount=100&crash=false";

    HttpClient client = HttpClient.newBuilder()
      .connectTimeout(Duration.ofSeconds(2))
      .build();

    // Use either platform threads or virtual threads.
    // Platform-thread option:
    // ExecutorService exec = Executors.newFixedThreadPool(attackers);

    // Java 21 virtual-thread option (recommended):
    ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();

    CountDownLatch start = new CountDownLatch(1);
    List<Future<String>> results = new ArrayList<>();

    for (int i = 0; i < attackers; i++) {
      results.add(exec.submit(() -> {
        start.await(); // synchronize start so requests collide
        HttpRequest req = HttpRequest.newBuilder()
          .uri(URI.create(url))
          .timeout(Duration.ofSeconds(10))
          .POST(HttpRequest.BodyPublishers.noBody())
          .build();
        HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
        return resp.statusCode() + " " + resp.body();
      }));
    }

    System.out.println("Launching " + attackers + " simultaneous transfers...");
    start.countDown();

    for (Future<String> f : results) {
      System.out.println(f.get());
    }

    exec.shutdown();
    exec.awaitTermination(30, TimeUnit.SECONDS);
  }
}
```

### Step 3: Run it and inspect the DB

Run your Spring Boot app, then run `AttackDoubleSpend`.

Expected “bad” behavior (without locking):

- Multiple requests report “success”
- DB ends up inconsistent (Account B credited multiple times even though A only had $100)

---

## Block 2 — Optimistic locking (`@Version`)

Optimistic locking assumes collisions are rare.

- It **does not lock** the row.
- It adds a `version` column and checks: “Did somebody else update this row since I read it?”
- If yes, the transaction fails at commit time with an optimistic locking exception.

### Step 1: Add a version column

PostgreSQL:

```sql
ALTER TABLE financial_accounts
ADD COLUMN version BIGINT NOT NULL DEFAULT 0;
```

### Step 2: Add `@Version` to the entity

In `Account`:

```java
import jakarta.persistence.Version;

@Version
private Long version;
```

### Step 3: Run the attack again

Now the concurrent requests should behave differently:

- One request succeeds
- The others fail with something like an `ObjectOptimisticLockingFailureException`

That is the database saying: “I refuse to accept your write because your copy is stale.”

### Step 4 (challenge): return HTTP 409 instead of a generic error

Right now your controller returns a plain string. For a real API, we want proper HTTP semantics:

- **200 OK** for success
- **409 Conflict** when a concurrent update prevented the transfer
- **400 Bad Request** (or **409**) for insufficient funds

**Option A: simple `try/catch` in the controller**

```java
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

@PostMapping("/transfer")
public ResponseEntity<String> executeTransfer(
    @RequestParam String from,
    @RequestParam String to,
    @RequestParam int amount,
    @RequestParam(defaultValue = "false") boolean crash) {

  try {
    transferService.transferMoney(from, to, amount, crash);
    return ResponseEntity.ok("Transfer of $" + amount + " successful!");
  } catch (OptimisticLockingFailureException e) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
      .body("Transfer failed: Account balance was updated by another transaction. Please try again.");
  } catch (IllegalStateException e) {
    return ResponseEntity.status(HttpStatus.CONFLICT).body("Transfer failed: " + e.getMessage());
  } catch (Exception e) {
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
      .body("Transfer failed and rolled back! Reason: " + e.getMessage());
  }
}
```

**Option B (cleaner): `@RestControllerAdvice`**

This keeps controllers clean and standardizes errors across endpoints.

---

## Block 3 — Pessimistic locking (`SELECT … FOR UPDATE`)

Pessimistic locking assumes collisions are likely (e.g., financial ledgers, ticket inventory).

- The first transaction **locks the row**.
- Other transactions **wait** until the lock is released.
- When they finally read, they see the latest state and your business rules (like “insufficient funds”) block invalid transfers.

### Step 1: Temporarily remove `@Version`

We want to test pessimistic locking without mixing strategies. Comment out `@Version` and the `version` field for this block (you can put it back later).

### Step 2: Add a locked “for update” repository method

In `AccountRepository`:

```java
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface AccountRepository extends JpaRepository<Account, Long> {
  Optional<Account> findByAccountName(String accountName);

  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("select a from Account a where a.accountName = :name")
  Optional<Account> findByAccountNameForUpdate(@Param("name") String name);
}
```

Hibernate will translate this into a row lock (PostgreSQL will use `FOR UPDATE` under the hood).

### Step 3: Update the service to use the locked reads

```java
@Transactional
public void transferMoney(String fromAccount, String toAccount, int amount, boolean simulateCrash) {
  Account sender = accountRepository.findByAccountNameForUpdate(fromAccount)
    .orElseThrow(() -> new RuntimeException("Sender not found"));
  Account receiver = accountRepository.findByAccountNameForUpdate(toAccount)
    .orElseThrow(() -> new RuntimeException("Receiver not found"));

  if (sender.getBalance() < amount) {
    throw new IllegalStateException("Insufficient funds");
  }

  sender.setBalance(sender.getBalance() - amount);
  accountRepository.save(sender);

  if (simulateCrash) {
    throw new RuntimeException("CRITICAL ERROR! Server crashed mid-transfer!");
  }

  receiver.setBalance(receiver.getBalance() + amount);
  accountRepository.save(receiver);
}
```

### Step 4: Run the attack again

Expected result with pessimistic locking:

- Requests don’t “race” anymore; they **queue** behind the DB lock.
- The first transfer succeeds.
- The next requests read the updated balance (0) and hit your `Insufficient funds` guard.
- No double spend.

This is powerful because it makes correctness “automatic” under contention—but the tradeoff is reduced throughput when many requests contend for the same row.

---

## Optimistic vs. pessimistic — when to use which

### Optimistic locking is best when…

- You have **high reads** and **low write collisions**
- You want **maximum throughput**
- You can tolerate occasional retries (clients can retry on 409)

Examples: profile updates, preferences, CMS edits.

### Pessimistic locking is best when…

- Collisions are common or extremely costly
- Correctness beats throughput
- You’re modeling “inventory” / “ledger” style resources

Examples: money transfers, seat reservations, ticketing, limited-stock checkout.

---

## The senior takeaway

`@Transactional` gives you atomicity and rollback. It does **not** automatically solve concurrent write anomalies.

To protect correctness under real traffic, you need **concurrency control**:

- **Optimistic**: detect conflicts, fail fast, retry.
- **Pessimistic**: prevent conflicts by locking.

This is exactly where backend engineering starts to feel “real”: not just writing endpoints, but protecting invariants under load.

Next: I want to tighten the API design (request/response bodies instead of query params), add validation, and introduce proper error models—still keeping correctness guarantees.

