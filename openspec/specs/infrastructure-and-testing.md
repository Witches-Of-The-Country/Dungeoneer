---
id: "infrastructure-and-testing"
title: "Infrastructure, CI/CD & Testing Strategy — Living Specification"
version: "1.0.0"
status: "active"
owners:
  - "@jaca"
scope:
  paths:
    - "docker-compose.yml"
    - ".github/workflows/**"
    - "**/Dockerfile"
    - "**/pom.xml"
resources:
  - "openspec/specs/sdd-stack-contract.md"
  - "openspec/specs/system-architecture.md"
---

# Infrastructure, CI/CD & Testing Strategy — Living Specification

> This is the authoritative source of truth for Dungeoneer's Docker Compose topology, repository structure, CI/CD pipelines, deployment targets, TDD methodology, and JaCoCo coverage enforcement.
> Update this file only after merging a completed change from `openspec/changes/`.

---

## Context & Objectives

Dungeoneer uses a polyglot orchestrator repository with Docker Compose for deterministic local development, GitHub Actions for per-service CI quality gates, Vercel + Railway for cloud deployment, and strict Test-Driven Development (TDD: Red → Green → Refactor) enforced by automated coverage thresholds.

---

## Functional Requirements

### [RF-01] Local Development Topology (`docker-compose.yml`)

All services run on a single bridge network (`dungeoneer-net`) with persistent PostgreSQL storage (`db-data` volume):

| Service | Image / Build Context | Port | Depends On |
|---------|----------------------|------|------------|
| `db` (`dungeoneer-db`) | `postgres:15-alpine` | `5432` | — |
| `player-character-service` | `./player-character-service/backend` | `8080` | `db` |
| `auth-service` | `./auth-service/backend` | `8082` | `db` |
| `api-gateway` | `./api-gateway/backend` | `9090` | `player-character-service`, `auth-service` |
| `dungeoneer-frontend` | `./dungeoneer-frontend` | `3000` | `api-gateway` |
| `campaign-service` *(Phase 2)* | `./campaign-service/backend` | `8081` | `db` |

### [RF-02] Environment Variable Contracts

Secrets are never hardcoded. Services consume environment variables injected via `.env` (local) or platform dashboards (production):

| Variable | Service | Default (dev only) |
|----------|---------|-------------------|
| `DB_PASSWORD` | `db`, backend services | *(none — must be set in `.env`)* |
| `SPRING_DATASOURCE_URL` | backend services | `jdbc:postgresql://db:5432/dungeoneer` |
| `SPRING_DATASOURCE_USERNAME` | backend services | `postgres` |
| `SPRING_DATASOURCE_PASSWORD` | backend services | `${DB_PASSWORD}` |
| `SPRING_PROFILES_ACTIVE` | all Spring services | `dev` |
| `NEXT_PUBLIC_API_URL` | `dungeoneer-frontend` | `http://localhost:9090` |

### [RF-03] Repository & Submodule Structure

```text
Dungeoneer/                          # Orchestrator monorepo
├── docker-compose.yml               # Canonical compose file
├── .env.example                     # Template for env vars
├── .github/workflows/               # Per-service CI pipelines
├── .agents/                         # Agent skills, rules, subagents
├── openspec/                        # Canonical living specs & active changes
├── AGENTS.md                        # Agent bootstrap file
├── dungeoneer-frontend/             # Next.js 16 application
├── player-character-service/        # Spring Boot 4.x character service
├── auth-service/                    # Spring Boot 4.x authentication service
├── campaign-service/                # Spring Boot 4.x campaign service (Phase 2)
└── api-gateway/                     # Spring Cloud Gateway service
```

### [RF-04] Deployment Architecture

| Environment | Frontend | Backend Services | Database |
|-------------|----------|------------------|----------|
| **Development** | `npm run dev` or Docker (`:3000`) | Docker Compose | PostgreSQL 15 in Docker (`:5432`) |
| **Production** | Vercel (`NEXT_PUBLIC_API_URL` → Railway Gateway) | Railway (separate service per microservice) | Railway PostgreSQL (shared instance, schema-per-service) |

### [RF-05] GitHub Actions CI/CD Workflows

- **Backend CI (`backend-ci.yml`)**: Triggers on push/PR to `main` affecting `player-character-service/**`. Sets up Java 21 (Temurin), caches Maven dependencies, runs `mvn clean test` (with Testcontainers), generates JaCoCo report, fails if domain + application line coverage < 80%, and verifies Docker build.
- **Frontend CI (`frontend-ci.yml`)**: Triggers on push/PR to `main` affecting `dungeoneer-frontend/**`. Sets up Node.js 20, runs `npm ci`, `npm run lint`, `npm run build`, `npx vitest run`, and `npx playwright test`.

---

## Non-Functional Requirements

### [RNF-01] TDD Workflow (Red → Green → Refactor)

Every feature or bugfix must follow the 3-step TDD cycle reflected in git commits:
1. **Red**: Write a failing test describing the expected behavior (`test: add failing test for ...`).
2. **Green**: Write the minimum production code required to pass the test (`feat: implement ...`).
3. **Refactor**: Clean up code while keeping all tests green (`refactor: extract ...`).

### [RNF-02] Test Pyramid & Coverage Targets

| Layer | Tool | Scope | Coverage Target |
|-------|------|-------|----------------|
| **Backend Unit** | JUnit 5 + Mockito (no Spring context) | Domain models, value objects, use case services, DTO validation | **≥ 80% line coverage** |
| **Backend Integration** | Testcontainers + PostgreSQL 15 | JPA repository adapters, Flyway migrations, custom queries | All repository methods |
| **Backend API** | Spring MockMvc | REST controllers, Jakarta Validation, RFC 7807 `ProblemDetail` responses | All endpoints |
| **Frontend Unit** | Vitest + React Testing Library | React components, hooks, utilities, form validation | **≥ 70% line coverage** |
| **Frontend E2E** | Playwright | End-to-end user flows (wizard, level-up, spell search) | Critical paths |

### [RNF-03] JaCoCo Maven Enforcement Configuration

Backend `pom.xml` files enforce the 80% coverage gate during `mvn verify`:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                        <includes>
                            <include>com.dungeoneer.*.domain.**</include>
                            <include>com.dungeoneer.*.application.**</include>
                        </includes>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## Out of Scope

- Kubernetes / Helm orchestration (Docker Compose + Railway/Vercel are used instead).

---

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0.0 | 2026-10-29 | @jaca | Migrated from `docs/spec/res/infrastructure.md` and `docs/spec/res/testing-strategy.md` |
