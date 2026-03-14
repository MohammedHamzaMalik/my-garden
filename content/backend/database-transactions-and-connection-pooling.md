---
title: Database Transactions & Connection Pooling — Plain Java, No Spring
date: 2026-02-28
tags:
  - java
  - database
  - jdbc
  - hikaricp
  - transactions
  - backend
description: Raw JDBC, HikariCP, and ACID transactions—transferring money between accounts, crashing halfway through, and watching the database roll back.
---

# Database Transactions & Connection Pooling — Plain Java, No Spring

After [[backend/dynamic-jvm-plugin-engine]] (how the JVM loads code) and [[backend/java-concurrency-from-race-condition-to-virtual-threads]] (how it runs many things at once), the next step was obvious: connect those threads to a **database**. And do it without Spring Boot—just plain Java and JDBC—so I could see exactly what happens under the hood.

---

## The problem: databases and concurrency

If a million virtual threads all try to open a new TCP connection to PostgreSQL at once, the database will crash. Connections are expensive. And if two threads update the same row at the same time, you get the same kind of race condition we saw in memory—but now with real money.

Two things solve this:

1. **Connection pooling** — Keep a small pool of pre-opened connections. Threads borrow one, run their query, and return it. No explosion of connections.
2. **ACID transactions** — Group multiple SQL statements into one atomic unit. If anything fails halfway through, the database rolls back everything as if it never happened.

---

## The task: atomic transfer, intentional crash

The exercise was:

1. Create a `financial_accounts` table with two rows: Account A ($1000) and Account B ($1000).
2. Write a plain Java program (no Spring) that transfers $500 from A to B.
3. **Intentionally crash** between the two `UPDATE` statements—after deducting from A, but before crediting B.
4. Use `connection.rollback()` and verify that Account A still has $1000. No money lost.

---

## Setup: database, Maven, HikariCP

**1. SQL (PostgreSQL):**

```sql
CREATE TABLE financial_accounts (
    id SERIAL PRIMARY KEY,
    account_name VARCHAR(50) NOT NULL,
    balance INT NOT NULL
);

INSERT INTO financial_accounts (account_name, balance) VALUES ('Account A', 1000);
INSERT INTO financial_accounts (account_name, balance) VALUES ('Account B', 1000);
```

**2. Maven dependencies (`pom.xml`):**

```xml
<dependencies>
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.2</version>
    </dependency>
</dependencies>
```

HikariCP is the default connection pool in Spring Boot. It keeps a pool of ready connections instead of opening a new one for every request.

---

## The code: transfer, crash, rollback

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class TransactionDemo {

    public static void main(String[] args) {
        // 1. Configure HikariCP — pool of 5 connections, ready to go
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/your_database_name");
        config.setUsername("your_username");
        config.setPassword("your_password");
        config.setMaximumPoolSize(5);

        HikariDataSource dataSource = new HikariDataSource(config);

        try (Connection connection = dataSource.getConnection()) {

            // 2. THE KEY: Turn off auto-commit
            // By default, every SQL statement commits immediately.
            // We group both UPDATEs into one transaction.
            connection.setAutoCommit(false);

            try {
                // Step A: Deduct $500 from Account A
                String deductSql = "UPDATE financial_accounts SET balance = balance - 500 WHERE account_name = 'Account A'";
                try (PreparedStatement stmt = connection.prepareStatement(deductSql)) {
                    stmt.executeUpdate();
                }

                // THE TRAP: Simulated crash — server loses power before crediting B
                throw new RuntimeException("Simulated Server Crash!");

                // Step B: Credit $500 to Account B (never reached)
                // String creditSql = "UPDATE financial_accounts SET balance = balance + 500 WHERE account_name = 'Account B'";
                // ...

            } catch (Exception e) {
                // 3. ROLLBACK — undo the deduction
                connection.rollback();
                System.out.println("Rolled back. Account A still has $1000.");
            }

        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            dataSource.close();
        }
    }
}
```

---

## What happens when you run it

1. The first `UPDATE` deducts $500 from Account A.
2. The `RuntimeException` is thrown before the second `UPDATE`.
3. `connection.rollback()` tells the database to undo the deduction.
4. You check the table: Account A still has $1000. No money disappeared.

That’s the “aha” moment: the database protects the data. Even though Java ran the first `UPDATE`, the transaction was never committed, so the change was discarded.

---

## What Spring Boot does for you

In a real Spring Boot app, you don’t write `setAutoCommit(false)`, `try/catch`, and `rollback()` yourself. You use `@Transactional`:

```java
@Transactional
public void transfer(int amount, String from, String to) {
    // Spring grabs a HikariCP connection, turns off auto-commit,
    // runs this method, and if any exception is thrown, it rolls back.
    accountRepository.deduct(from, amount);
    accountRepository.credit(to, amount);
}
```

Spring does the same thing we did manually: borrow a connection, disable auto-commit, run the logic, and roll back on any exception. Understanding the manual version makes `@Transactional` feel less like magic.

---

## Where this fits in the roadmap

So far the path looks like this:

| Pillar | What I learned |
|--------|----------------|
| **1. JVM Architecture** | How code is loaded (ClassLoaders, [[backend/dynamic-jvm-plugin-engine]]) |
| **2. Concurrency** | How to run many tasks at once ([[backend/java-concurrency-from-race-condition-to-virtual-threads]]) |
| **3. Database Integrity** | How to protect data during failures (Connection pools, ACID, rollback) |

Next step: take this transfer logic and expose it as a REST API with Spring Boot—so a real HTTP request can trigger the transaction. That’s the next post on my [[backend/java-backend-roadmap]].
