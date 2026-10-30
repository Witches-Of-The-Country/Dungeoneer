# Dungeoneer — Agent Guidelines

> This file bootstraps AI agents working on the Dungeoneer codebase.  
> Read this file FIRST before making any changes.

---

## Project Overview

Dungeoneer is a D&D 5e/2024 companion web app for managing character sheets, campaigns, and game sessions. It uses a microservices architecture with Clean Architecture principles.

## Specifications & Knowledge Index

| Document | Path | Load When |
|----------|------|-----------|
| **SDD Stack Contract** | `openspec/specs/sdd-stack-contract.md` | Always — technology stack, build/test commands, database, and architectural rules |
| **System Architecture** | `openspec/specs/system-architecture.md` | Always — platform vision, C4 diagrams, Hexagonal package rules, Dual Ruleset strategy, and gateway routing |
| **Player Character Domain & API** | `openspec/specs/player-character-domain.md` | Working on character domain models, GoF patterns, use cases (UC1–UC5), SRD scope, or REST endpoints |
| **Player Character Data Model** | `openspec/specs/player-character-data-model.md` | Working on `player_character` PostgreSQL schema, ER diagrams, tables, constraints, or Flyway migrations |
| **Campaign Domain** | `openspec/specs/campaign-domain.md` | Working on Phase 2/3 campaigns, NPCs, encounters (UC6–UC7), or `campaign` schema |
| **Frontend Web App** | `openspec/specs/frontend-web-app.md` | Working on Next.js 16 pages, Dark Fantasy RPG UI tokens, Wizard, Character Sheet Dashboard, or TanStack Query |
| **Infrastructure & Testing** | `openspec/specs/infrastructure-and-testing.md` | Working on Docker Compose, CI/CD pipelines, deployment, TDD workflow, or JaCoCo coverage |
| **README** | `README.md` | Domain model class diagrams and design decisions (pt-BR overview) |

## Core Architectural Patterns

### 1. Clean Architecture (Hexagonal)

Every Spring Boot service follows this package structure:

```
domain/model/         → Entities, Value Objects, Domain Interfaces (NO framework imports)
application/port/in/  → Use Case interfaces (inbound ports)
application/port/out/ → Repository port interfaces (outbound ports)  
application/service/  → Use Case implementations
application/dto/      → Data Transfer Objects (with Jakarta Validation)
adapter/in/web/       → REST Controllers + Web Mappers
adapter/out/persistence/ → JPA Entities + Repository Implementations
```

**Golden Rule**: The `domain` package MUST have ZERO framework imports. No Spring, no JPA, no Lombok on domain classes.

### 2. Dual Ruleset Engine

All reference data (classes, spells, feats, lineages, backgrounds) is tagged with a `ruleset` discriminator (`SRD_2014` | `SRD_2024` | `HOMEBREW`). The `RulesEngine` interface (Strategy Pattern) resolves mechanics based on the character's declared ruleset.

### 3. Data Ownership

- **player-character-service** owns: characters, ability scores, spells, feats, items, lineages, backgrounds, archetypes, subclasses
- **campaign-service** owns: campaigns, NPCs, encounters, and the campaign↔character association (stores `characterId` references, hydrates via REST)
- Services NEVER cross-query another service's database schema

### 4. Error Handling

All REST APIs return **RFC 7807 ProblemDetail** responses via `@ControllerAdvice`. Domain exceptions map to specific HTTP status codes.

### 5. Testing

**TDD (Red → Green → Refactor)**. JaCoCo enforces ≥80% line coverage on domain + application layers.

| Layer | Tool |
|-------|------|
| Unit | JUnit 5 + Mockito |
| Integration | Testcontainers + PostgreSQL |
| API | MockMvc |
| Frontend Unit | Vitest + Testing Library |
| Frontend E2E | Playwright |

## Conventions

| Aspect | Convention |
|--------|-----------|
| Language (code) | English |
| Language (UI) | Portuguese (pt-BR) |
| Branch strategy | Feature branches → PR → main |
| Commit style | Conventional Commits |
| API documentation | OpenAPI 3.0 via springdoc |
| Database migrations | Flyway (SQL-based, numbered) |
| IDs | UUID (gen_random_uuid) |
| Timestamps | TIMESTAMPTZ (UTC) |
| DTO validation | Jakarta Validation annotations |
| Build tool (Java) | Maven (global `mvn`, not wrapper) |
| Build tool (JS) | npm |

## Security Notes

- **MVP has NO authentication**. SecurityConfig permits all.
- `user_id` columns exist as NULLABLE FK placeholders.
- NEVER hardcode secrets. Use environment variables.
- CORS is configured at the API Gateway level only.

## Infrastructure

- **Local dev**: Docker Compose (single `docker-compose.yml` at repo root)
- **Production**: Vercel (frontend) + Railway (backend + PostgreSQL)
- **Database**: PostgreSQL 15, shared instance, schema-per-service
- **CI/CD**: GitHub Actions per service

## Agent Workflow & Commits

- **Ready-to-use Commit Commands**: Whenever a task or logical chunk of work is finished, the agent MUST provide the exact, ready-to-copy-and-paste `git commit` command (following the Conventional Commits style) for the user to execute.
- **Phase Pauses**: When following multi-phase implementation plans (such as TDD cycles or moving between different microservices/components), the agent MUST pause and prompt the user to commit their current progress before proceeding to the next phase. Always provide the relevant commit command during these pauses.
