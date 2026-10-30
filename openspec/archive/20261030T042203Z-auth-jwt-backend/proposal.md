# Proposal: Auth Jwt Backend

> **Change ID:** `auth-jwt-backend`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Implement user registration (`POST /api/auth/register`) and authentication (`POST /api/auth/login`) in `auth-service` following Clean Architecture principles. Users are persisted in PostgreSQL with BCrypt-hashed passwords, and successful authentication issues a signed HMAC-SHA256 JWT token containing the user's UUID in the `sub` claim (and `email`/`username` claims) so downstream services (`player-character-service` via `api-gateway`) can associate and authorize player characters by `user_id`.

## Motivation

Every player character in Dungeoneer must belong to an authenticated user (`user_id`). Currently, `auth-service` only contains a basic `UserJpaEntity`, `UserRepository`, and `V1__create_users_table.sql` migration, with no domain model, use cases, REST controllers, password hashing, or JWT token issuance. Completing `auth-service` unblocks user-scoped character creation, listing, editing, and deletion.

## Scope

### In Scope
- Refactor `auth-service` into full Hexagonal / Clean Architecture (`domain/model`, `application/port/in`, `application/port/out`, `application/service`, `application/dto`, `adapter/in/web`, `adapter/out/persistence`, `core/config`, `core/web`).
- Implement `User` domain model (zero framework imports) with `id`, `email`, `nickname`, `birthdate`, `passwordHash`, and `roles` invariants matching `V1__create_users_table.sql`.
- Fix `AuthServiceApplication` package scanning (`@SpringBootApplication(scanBasePackages = "com.dungeoneer.auth")`).
- Implement `RegisterUserUseCase` (`POST /api/auth/register` → `201 Created`) and `LoginUserUseCase` (`POST /api/auth/login` → `200 OK`), both returning an `AuthResponseDto` containing `accessToken`, `tokenType` (`Bearer`), `expiresIn`, and `user` summary (`id`, `nickname`, `email`, `roles`).
- Implement `PasswordEncoderPort` (backed by Spring Security `BCryptPasswordEncoder`) and `TokenProviderPort` (backed by Nimbus JOSE + JWT / `spring-boot-starter-oauth2-resource-server` or `javax.crypto.Mac` HMAC-SHA256 using `${JWT_SECRET}`).
- Configure `SecurityConfig` in `auth-service` (`stateless`, CSRF disabled, `/api/auth/**` and `/actuator/health` permitted).
- Implement `GlobalExceptionHandler` returning RFC 7807 `ProblemDetail` for duplicate email/username (`409 Conflict`), invalid credentials (`401 Unauthorized`), and Jakarta validation errors (`422 Unprocessable Entity`).
- Comprehensive TDD test suite: Domain unit tests, Application service unit tests (Mockito), Persistence integration tests (Testcontainers PostgreSQL 15), and Controller MockMvc tests (≥80% JaCoCo coverage).

### Out of Scope
- OAuth2 social login, password reset emails, or refresh token rotation (deferred to future auth enhancements).
- Gateway JWT routing filter (covered in `gateway-routing-and-cors`).

## Proposed Solution

### Approach
`auth-service` will expose `/api/auth/register` and `/api/auth/login`. When registering, `RegisterUserService` normalizes the email, verifies uniqueness of `email` and `username` via `UserRepositoryPort`, hashes the raw password via `PasswordEncoderPort`, persists the new `User`, and generates a signed JWT via `TokenProviderPort`. When logging in, `LoginUserService` looks up the user by email, verifies the password hash, and issues a signed JWT with `sub = user.getId().toString()`, `iat`, and `exp` (configurable via `${JWT_EXPIRATION_SECONDS:86400}`).

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `auth-service/backend/pom.xml` | Modify | Add `spring-boot-starter-security`, `spring-boot-starter-validation`, `spring-boot-starter-oauth2-resource-server` (for Nimbus JWT), `jacoco-maven-plugin`, and `testcontainers` |
| `auth-service/backend/src/main/resources/application.properties` | Modify | Configure `jwt.secret=${JWT_SECRET}` and `jwt.expiration-seconds=${JWT_EXPIRATION_SECONDS:86400}` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java` | Create | Pure Java domain entity with invariants |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/exception/DuplicateUserException.java` | Create | Domain exception for email/username collision (`409`) |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/exception/InvalidCredentialsException.java` | Create | Domain exception for bad login (`401`) |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/RegisterUserUseCase.java` | Create | Inbound port for user registration |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/LoginUserUseCase.java` | Create | Inbound port for user login |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/UserRepositoryPort.java` | Create | Outbound port for user persistence |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/PasswordEncoderPort.java` | Create | Outbound port for password hashing |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/out/TokenProviderPort.java` | Create | Outbound port for JWT signing |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/service/AuthService.java` | Create | Application service implementing registration and login |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java` | Create | REST controller for `/api/auth/register` and `/api/auth/login` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserJpaEntity.java` | Modify | Align with `User` domain mapping |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserPersistenceAdapter.java` | Create | Implements `UserRepositoryPort` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/security/JwtTokenProviderAdapter.java` | Create | HMAC-SHA256 JWT signer using `${JWT_SECRET}` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/security/BCryptPasswordEncoderAdapter.java` | Create | BCrypt implementation of `PasswordEncoderPort` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/core/config/SecurityConfig.java` | Create | Stateless Spring Security configuration |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/core/web/GlobalExceptionHandler.java` | Create | RFC 7807 `ProblemDetail` error handler |

### Data Model Changes (if applicable)
Uses existing `users` table in `auth-service/backend/src/main/resources/db/migration/V1__create_users_table.sql` (`id UUID PK`, `email VARCHAR(255) UNIQUE NOT NULL`, `nickname VARCHAR(255) NOT NULL`, `birthdate DATE NOT NULL`, `password_hash VARCHAR(255) NOT NULL`, `roles VARCHAR[] NOT NULL`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`).

## Acceptance Criteria

- [ ] **Given** a valid registration payload (`username`, `email`, `password` ≥ 8 chars) **When** `POST /api/auth/register` is called **Then** the user is persisted with a BCrypt password hash and `201 Created` is returned with a signed JWT (`sub` = user UUID) and user profile DTO.
- [ ] **Given** an already registered `email` or `username` **When** `POST /api/auth/register` is called **Then** `409 Conflict` is returned as an RFC 7807 `ProblemDetail`.
- [ ] **Given** valid user credentials (`email`, `password`) **When** `POST /api/auth/login` is called **Then** `200 OK` is returned with a valid HMAC-SHA256 JWT token whose `sub` claim equals the user's UUID.
- [ ] **Given** an unknown email or wrong password **When** `POST /api/auth/login` is called **Then** `401 Unauthorized` is returned as an RFC 7807 `ProblemDetail` without leaking whether the email exists.

## Defensive Security & Validation Considerations

### Input Validation
- `RegisterRequest`: `@NotBlank @Size(min = 3, max = 50) username`, `@NotBlank @Email email`, `@NotBlank @Size(min = 8, max = 100) password`.
- `LoginRequest`: `@NotBlank @Email email`, `@NotBlank password`.

### Secret Isolation
- `JWT_SECRET` is injected strictly via environment variables (`${JWT_SECRET}`). Startup fails fast if `JWT_SECRET` is missing or shorter than 32 bytes (256 bits).
- Passwords are never logged or returned in any DTO; only BCrypt hashes are stored.

### Authorization & Authentication Impact
- Establishes the canonical identity provider (`auth-service`) issuing JWTs consumed by `api-gateway`.

### Other Security Risks
- Generic `401 Unauthorized` error messages on login prevent user enumeration attacks.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Register new user with valid payload | Unit & API (MockMvc) | Returns `201 Created` with signed JWT and user info |
| TP-02 | Login existing user with valid password | Unit & API (MockMvc) | Returns `200 OK` with signed JWT containing `sub = userId` |
| TP-03 | Persist and query `User` via `UserPersistenceAdapter` | Integration (Testcontainers) | Saves and retrieves user by email/username accurately |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Register with duplicate email or username | Unit & API (MockMvc) | Throws `DuplicateUserException` → `409 Conflict` ProblemDetail |
| TE-02 | Login with wrong password or non-existent email | Unit & API (MockMvc) | Throws `InvalidCredentialsException` → `401 Unauthorized` ProblemDetail |
| TE-03 | Register with invalid email or short password (< 8 chars) | API (MockMvc) | Returns `422 Unprocessable Entity` ProblemDetail |

### Test Commands
- **Run tests:** `mvn -f auth-service/backend/pom.xml clean test`
- **Coverage check:** `mvn -f auth-service/backend/pom.xml clean verify`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Weak `JWT_SECRET` in dev/prod | Low | High | Validate secret length ≥ 32 bytes on bean initialization |

## Specification Changes & Delta Stubs

### `specs/system-architecture-delta.md`
- **Target Canonical Spec:** `openspec/specs/system-architecture.md`
- **Summary of Updates:** Update Security Posture (`[RNF-02]`) and API Gateway Routing (`[RF-03]`) from "MVP has no authentication" to JWT authentication via `auth-service` (`/api/auth/register`, `/api/auth/login`) and `X-User-Id` header propagation.

## References

- `openspec/specs/system-architecture.md`
- `openspec/specs/sdd-stack-contract.md`
