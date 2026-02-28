---
title: Dynamic JVM Plugin Engine — Hot-Swappable Spring Boot Backend
date: 2026-02-28
tags:
  - java
  - spring-boot
  - plugins
  - backend
  - project
description: A Spring Boot server that dynamically loads and runs external Java plugins at runtime with a custom ClassLoader—zero downtime, no restart.
---

# Dynamic JVM Plugin Engine

I built a **hot-swappable backend** that loads, compiles, and runs external Java code at runtime without restarting the server. Here’s what it does and how it works.

**Repo:** [MohammedHamzaMalik/plugin-server](https://github.com/MohammedHamzaMalik/plugin-server)

## What it is

Standard Java apps need every `.class` file on the classpath at startup. This project goes further: a **custom ClassLoader engine** that:

- Watches a directory for new plugin `.class` files
- Loads them into the JVM on the fly
- Exposes them via REST so you can trigger plugin execution without restarting

So you can drop a new plugin in, and the server picks it up and runs it—**zero-downtime extensibility**.

## Tech stack and mechanisms

| Area | Choice |
|------|--------|
| **Framework** | Spring Boot 3.x / Java 21 |
| **Loading** | Custom `java.lang.ClassLoader` |
| **Watching** | Java NIO `WatchService` |
| **Execution** | Java Reflection API |
| **Concurrency** | `CommandLineRunner`, background thread, `ConcurrentHashMap` |

## Architecture highlights

1. **ClassLoader isolation** — The custom ClassLoader loads raw bytecode from a plugin directory and uses the Spring Boot application ClassLoader as parent. That keeps external code compatible with your interfaces and avoids `ClassCastException` when passing plugin instances around.

2. **Non-blocking watch** — A dedicated background thread runs the file watcher so it never blocks the main Tomcat thread.

3. **Thread-safe registry** — Plugins are stored in a `ConcurrentHashMap`, so registration and lookup are safe under concurrent HTTP requests.

## How to run and test hot-swap

1. **Start the server**  
   `./mvnw spring-boot:run` — it creates a `./plugins` directory and watches it.

2. **Check loaded plugins**  
   `curl http://localhost:8080/api/plugins` → initially `[]`.

3. **Implement a plugin** (outside the project), e.g. `DataCleanupPlugin.java` implementing `TaskPlugin`, with `getName()` and `execute()`.

4. **Compile** against the server’s classes:  
   `javac -cp path/to/plugin-server/target/classes DataCleanupPlugin.java`

5. **Drop** `DataCleanupPlugin.class` into `./plugins`. The console should log registration.

6. **Run via API**  
   `curl -X POST http://localhost:8080/api/plugins/DataCleanup/run`

## Why it matters for learning

This project ties together:

- **ClassLoaders** — how the JVM loads classes and how to load bytecode yourself
- **NIO WatchService** — reactive file/directory monitoring
- **Spring Boot** — embedding custom lifecycle (e.g. `CommandLineRunner`) and REST APIs
- **Concurrency** — safe shared state and background tasks

It’s a concrete step on my [[java-backend-roadmap]] and a good base for experimenting with plugin APIs, security boundaries, or versioning later.
