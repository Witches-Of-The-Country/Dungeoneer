# Delta Spec: `sdd-stack-contract`

> **Change ID:** `migrate-legacy-specs-to-openspec`
> **Target Canonical Spec:** `openspec/specs/sdd-stack-contract.md`
> **Change Type:** Modify

## Architectural Updates

Replace the placeholder template in `openspec/specs/sdd-stack-contract.md` with the complete Dungeoneer stack contract below:

---

# SDD Stack Contract

> This file declares the technology stack and architectural constraints of the Dungeoneer project.
> It is read by skills and subagents to make stack-aware decisions.

---

## Stack Identity

- **Project Name:** `dungeoneer` (Dungeoneer — D&D 5e/2024 Companion Platform)
- **Primary Language:** Java 21 (Backend microservices) & TypeScript 5.x (Frontend web application)
- **Runtime / Platform:** JVM 21 (Eclipse Temurin LTS) & Node.js 20 (LTS)
- **Framework:**
  - **Backend Services (`player-character-service`, `auth-service`, `campaign-service`):** Spring Boot 4.0.6 (Spring WebMVC, Spring Data JPA, Spring Security, Jakarta Validation)
  - **API Gateway (`api-gateway`):** Spring Cloud Gateway Server WebMVC (Spring Boot 4.x)
  - **Frontend (`dungeoneer-frontend`):** Next.js 16.2.6 (App Router), React 19.2.4, TailwindCSS 4.x, shadcn/ui + Radix UI, TanStack Query, React Hook Form + Zod
- **Package Manager:** Maven 3.9.6 (always use global `mvn`, never `./mvnw`) & `npm`

---

## Project Structure

- **Source Root:**
  - Backend: `<service>/backend/src/main/java/com/dungeoneer/` and `<service>/backend/src/main/resources/`
  - Frontend: `dungeoneer-frontend/app/`, `dungeoneer-frontend/components/`, `dungeoneer-frontend/lib/`
- **Test Root:**
  - Backend: `<service>/backend/src/test/java/com/dungeoneer/`
  - Frontend: `dungeoneer-frontend/__tests__/` (and colocated `*.test.ts(x)` / `e2e/`)
- **Entry Point:**
  - Backend: `com.dungeoneer.BackendApplication` (per Spring Boot service)
  - Frontend: `dungeoneer-frontend/app/layout.tsx` & `dungeoneer-frontend/app/page.tsx`
- **Build Output:**
  - Backend: `<service>/backend/target/`
  - Frontend: `dungeoneer-frontend/.next/`

---

## Test Runner

- **Command:**
  - Backend: `mvn clean test` (run inside `<service>/backend` or via `mvn -f <service>/backend/pom.xml clean test`)
  - Frontend Unit: `npx vitest run`
  - Frontend E2E: `npx playwright test`
- **Test Framework:**
  - Backend: JUnit 5 (Jupiter) + Mockito (Unit), Testcontainers + PostgreSQL 15 (Integration), Spring MockMvc (API)
  - Frontend: Vitest + React Testing Library (Component/Unit), Playwright (E2E)
- **Coverage Command:**
  - Backend: `mvn clean verify` (JaCoCo `0.8.12` enforces **≥ 80% line coverage** on `com.dungeoneer.*.domain.**` and `com.dungeoneer.*.application.**`)
  - Frontend: `npx vitest run --coverage` (**≥ 70% line coverage** on components)

---

## Code Style

- **Linter:** ESLint 9 (`eslint-config-next`) for Frontend (`npm run lint`); Java 21 compiler checks + Clean Architecture package boundary rules for Backend
- **Formatter:** Standard Java 21 / Spring conventions; Prettier / TailwindCSS 4 conventions for TypeScript/React
- **Format Command:** `npm run lint` (Frontend)

---

## API Contract (if applicable)

- **API Type:** REST over HTTP (JSON payloads; **RFC 7807 `ProblemDetail`** for all error responses via `@ControllerAdvice`)
- **Spec Location:** OpenAPI 3.0 auto-generated via `springdoc-openapi-starter-webmvc-ui` at `/v3/api-docs` and `/swagger-ui.html`; canonical REST contracts documented in `openspec/specs/player-character-domain.md` and `openspec/specs/campaign-domain.md`. Generated TypeScript client target: `dungeoneer-frontend/lib/api/generated/`.
- **Base URL (dev):**
  - Frontend: `http://localhost:3000`
  - API Gateway: `http://localhost:9090`
  - `player-character-service`: `http://localhost:8080`
  - `campaign-service` (Phase 2): `http://localhost:8081`
  - `auth-service`: `http://localhost:8082`

---

## Database (if applicable)

- **Engine:** PostgreSQL 15 (`postgres:15-alpine`, port `5432`, shared instance `dungeoneer` with schema-per-service isolation: `player_character`, `campaign`)
- **Migration Tool:** Flyway (`spring-boot-starter-flyway` + `flyway-database-postgresql`)
- **Migration Command:** SQL-based numbered migrations in `<service>/backend/src/main/resources/db/migration/V<N>__<description>.sql` (applied automatically on service startup or via `mvn flyway:migrate`)

---

## Notes

- **Clean Architecture Golden Rule:** Every backend service follows Hexagonal / Clean Architecture (`domain/model/`, `application/port/in/`, `application/port/out/`, `application/service/`, `application/dto/`, `adapter/in/web/`, `adapter/out/persistence/`). The `domain` package MUST have **ZERO** framework imports (no Spring, no JPA, no Jakarta Validation, no Lombok).
- **Dual Ruleset Engine:** All reference data is tagged with a `ruleset` discriminator (`SRD_2014` | `SRD_2024` | `HOMEBREW`) and resolved via the `RulesEngine` Strategy interface.
- **Data Ownership:** Services NEVER cross-query another service's database schema. Cross-service data is hydrated via REST through the API Gateway.
- **Language & Localization:** Code, API routes, DB schemas, comments, and git commits MUST be in **English**. All UI labels, text, and user-facing messages MUST be in **Portuguese (`pt-BR`)**.
- **Security:** Never hardcode secrets, passwords, or tokens — always use environment variable injection (`${DB_PASSWORD}`). Inbound DTOs must be validated using Jakarta Validation annotations. CORS is configured at the API Gateway level only.
