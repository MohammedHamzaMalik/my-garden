---
title: Deploying the Transfer API — Docker and Beyond
date: 2026-03-09
tags:
  - java
  - spring-boot
  - docker
  - deployment
  - backend
description: Containerize the transfer API with a multi-stage Dockerfile, run app + PostgreSQL with Docker Compose, and options for cloud deployment.
---

# Deploying the Transfer API — Docker and Beyond

After [[testing-the-transfer-api-unit-and-integration]], the transfer API had tests and a clear contract. The next step was to **ship it**: containerize with Docker so it runs the same everywhere, then wire it to a real database and point it at the internet.

This post covers a practical path from "runs on my machine" to "runs in a container (and optionally in the cloud)."

---

## Why Docker first?

- **Reproducible** — same Java version, same OS, same dependencies in dev and prod.
- **Portable** — run locally, on a VPS, or on any cloud that supports containers.
- **Layered** — build the JAR once, then run it in a minimal image; no need to install Java on the host.

---

## 1. Multi-stage Dockerfile for Spring Boot

A **multi-stage** build compiles in one image and runs in a smaller one. Stage 1: Maven + Java to build. Stage 2: only the JAR + a slim JRE.

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B

COPY src src
RUN ./mvnw package -DskipTests -B

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Notes:

- `eclipse-temurin:21-jdk-alpine` — Java 21 for the build.
- `eclipse-temurin:21-jre-alpine` — smaller JRE-only image for runtime.
- `go-offline` caches dependencies so rebuilds are faster when only source changes.
- Non-root user for a bit of hardening.

---

## 2. Docker Compose: app + PostgreSQL

For local or simple deployments, run the app and database together:

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: bank
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d bank"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/bank
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: secret
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Run:

```bash
docker compose up -d
```

The app waits for PostgreSQL to be ready, then starts. You’ll need to seed the `financial_accounts` table (e.g. via a migration or init script) before hitting `/api/transfer`.

---

## 3. Database schema and seed data

If you use Flyway or Liquibase, add a migration. For a quick manual setup, you can run SQL after the DB is up:

```sql
CREATE TABLE IF NOT EXISTS financial_accounts (
    id SERIAL PRIMARY KEY,
    account_name VARCHAR(50) NOT NULL,
    balance INT NOT NULL,
    version BIGINT NOT NULL DEFAULT 0
);

INSERT INTO financial_accounts (account_name, balance, version)
VALUES ('Account A', 1000, 0), ('Account B', 1000, 0)
ON CONFLICT DO NOTHING;
```

(Adjust if you use a different schema or skip `version` for pessimistic-only locking.)

---

## 4. Build and run locally

```bash
docker compose build
docker compose up -d
```

Then:

```bash
curl -X POST http://localhost:8080/api/transfer \
  -H "Content-Type: application/json" \
  -d '{"fromAccount":"Account A","toAccount":"Account B","amount":100,"simulateCrash":false}'
```

---

## 5. Cloud deployment options

Once the image runs locally, you can push it to a registry and deploy:

- **Railway / Render / Fly.io** — connect a GitHub repo, set env vars (DB URL, etc.), and they build + run the container. Often include managed PostgreSQL.
- **AWS (ECS, App Runner)** — push the image to ECR, define a task/service, and point it at RDS or another DB.
- **Google Cloud Run** — serverless containers; good fit for APIs with variable traffic.
- **VPS (DigitalOcean, Hetzner, etc.)** — run `docker compose` on a small VM; you manage the host and backups.

Common pattern: **managed DB** (e.g. Railway Postgres, AWS RDS) + **container for the app**, with `SPRING_DATASOURCE_URL` and credentials from environment variables.

---

## 6. What stays the same

- **Code** — no changes to the transfer logic, locking, or validation.
- **Tests** — unit and integration tests still run with `./mvnw test`; Docker is orthogonal.
- **Env-based config** — `application.properties` can use placeholders; override with env vars in the container.

---

## Where this sits in the roadmap

So far: [[spring-boot-transactional-rest-api]], [[why-transactional-isnt-enough-jpa-locking]], [[rest-api-design-dtos-validation-error-handling]], [[testing-the-transfer-api-unit-and-integration]], and now **deployment**—Docker, Compose, and a path to the cloud. The transfer API is built, tested, and shippable. Next on my [[java-backend-roadmap]]: more features (e.g. transfer history, auth) or deeper infra (CI/CD, monitoring).
