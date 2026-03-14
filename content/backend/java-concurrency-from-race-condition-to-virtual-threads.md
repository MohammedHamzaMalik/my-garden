---
title: Java Concurrency — How I Started With a Simple Loop and Ended With Virtual Threads
date: 2026-02-28
tags:
  - java
  - concurrency
  - backend
  - threads
  - java-21
description: From a sequential BankAccount loop to a race condition, then AtomicInteger, ExecutorService, and Java 21 Virtual Threads—why concurrency is about I/O, not "faster math."
---

# Java Concurrency — How I Started With a Simple Loop and Ended With Virtual Threads

After building the [[backend/dynamic-jvm-plugin-engine]], the next step was clear: stop spinning up raw threads and learn how the JVM runs *many* things at once. That meant **concurrency, thread pools, and virtual threads**. Here’s how I started, how I broke it, and how I fixed it.

---

## Where I started: a simple loop that was “correct”

I had a `BankAccount` and a loop that did a million deposits:

```java
public class BankAccount {
    public int balance = 0;

    public void deposit(int amount) {
        balance = balance + amount;
    }
}
```

```java
void main() {
    int n = 1000000;
    BankAccount account = new BankAccount();
    for (int i = 0; i < n; i++) {
        account.deposit(1);
    }
    System.out.println("total - " + account.balance);
}
```

It was **fast and correct** every time: `1000000`. No concurrency—just one thread doing one thing after another. So far so good.

---

## The exercise: make it fail on purpose

The goal was to *see* a race condition. So I changed the program to do a million deposits **concurrently**: one new thread per deposit.

```java
Thread[] threads = new Thread[n];
for (int i = 0; i < n; i++) {
    threads[i] = new Thread(() -> account.deposit(1));
    threads[i].start();
}
for (int i = 0; i < n; i++) {
    threads[i].join();
}
System.out.println("total - " + account.balance);
```

Result: the final balance was **wrong** every run—something like `998124` or `995001` instead of `1000000`.

**Why?** Because `balance = balance + amount` is not one atomic step. In the JVM it’s:

1. **Read** the current `balance`
2. **Add** `amount`
3. **Write** the result back

Two threads can both read `0`, both add 1, both write `1`—and one deposit is lost. With a million threads, thousands of updates overwrite each other. That’s the race condition.

Seeing that wrong number was the moment it clicked: you can’t treat a plain `int` as safe when many threads touch it.

---

## Fix 1: thread-safe state with `AtomicInteger`

Locking the whole method with `synchronized` would work, but we used something better: **`AtomicInteger`** from `java.util.concurrent.atomic`.

It uses **Compare-And-Swap (CAS)** at the CPU level: “Has this value changed since I read it? If no, update it; if yes, retry.” One hardware-level step, no explicit lock, and still thread-safe.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class BankAccount {
    public AtomicInteger balance = new AtomicInteger(0);

    public void deposit(int amount) {
        balance.addAndGet(amount);  // single, indivisible operation
    }
}
```

Same million concurrent deposits, but the final balance is always **exactly** `1000000`.

---

## Fix 2: don’t spawn a million OS threads — use a thread pool

A million real (platform) threads would be a disaster: each one costs a lot of memory and OS resources. So we stopped creating threads by hand and used an **`ExecutorService`**: a fixed pool of workers that process a queue of tasks.

Think of it as 10 tellers and a line of customers. You submit a million “deposit 1” tasks; 10 threads keep taking work from the queue until everything is done.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < n; i++) {
    executor.submit(() -> account.deposit(1));
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.MINUTES);

System.out.println("total - " + account.balance.get());
```

So we fixed two things:

1. **Correctness** — `AtomicInteger` so no lost updates.
2. **Resource usage** — a small, fixed number of threads instead of a million.

---

## “My simple loop was fast and correct. Why all this complexity?”

The key insight: **concurrency isn’t mainly about making pure math faster. It’s about not blocking.**

My original loop was fast because it was only CPU and RAM—no waiting. Adding threads to that kind of work often *slows* things down (context switching, lock contention). So for “just add 1 a million times,” the simple loop is the right tool.

In a real backend, `deposit` isn’t just math. It’s more like:

1. Receive the request
2. Open a connection to the database
3. **Wait** for the DB (e.g. 50 ms)
4. Update the ledger

If that wait is 50 ms per deposit and you do it **sequentially** in one thread:

- 1,000 deposits × 50 ms = **50 seconds**, and the server does nothing else while waiting.

If you do it **concurrently** with a thread pool:

- While one thread is blocked on the DB, others can run. Same 1,000 deposits in a small number of seconds, and the server can still handle other requests.

So we need thread pools and concurrency not because addition is slow, but because **networks, databases, and APIs are slow**. We use a limited pool of threads so that while some requests are waiting on I/O, the CPU can work on others.

---

## Upgrade: Java 21 virtual threads

Platform threads are heavy (~1 MB each). You can’t afford 10,000 of them just to have 10,000 requests waiting on the DB. So the classic approach is exactly what we did: a small pool (e.g. 10–100 threads) and a queue.

**Java 21 (Project Loom)** changes the game with **virtual threads**: lightweight threads managed by the JVM, not the OS. They use very little memory, so you can have huge numbers of them.

Switching from a fixed pool to “one virtual thread per task” was a one-line change:

```java
// Before: fixed pool of 10 platform threads
ExecutorService executor = Executors.newFixedThreadPool(10);

// After: one new virtual thread per submitted task
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

With that, we effectively ran **one million threads** (virtual) at once. The balance stayed correct, and the JVM handled it without blowing up. That’s the kind of scale modern Java is aiming for.

Spring Boot 3.2+ can run the whole web stack on virtual threads with a single setting: `spring.threads.virtual.enabled=true`. So the same ideas (don’t block, scale with many concurrent I/O-bound tasks) now apply at the framework level.

---

## How I started and how I ended

| Stage | What I had | What I learned |
|--------|------------|----------------|
| **Start** | Simple loop, one thread, correct and fast | Concurrency is not “more threads = faster.” |
| **Break it** | Million threads, shared `int`, wrong balance | Race conditions: read–modify–write is not atomic. |
| **Fix state** | `AtomicInteger` + CAS | Thread-safe updates without a big lock. |
| **Fix scale** | `ExecutorService` with fixed pool | Reuse threads; don’t create millions of OS threads. |
| **Why** | “Why not just the simple loop?” | Concurrency is for **blocking I/O** (DB, network), not for making pure math faster. |
| **Modern** | `newVirtualThreadPerTaskExecutor()` | Huge number of concurrent tasks without the cost of platform threads. |

This is the kind of progression that separates “I can write a loop” from “I can reason about a server under load.” Next step in my [[backend/java-backend-roadmap]] is to keep building on this—more concurrent patterns and then layering in persistence and APIs.
