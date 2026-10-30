# Tasks: Auth Jwt Backend

> **Change ID:** `auth-jwt-backend`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Configure `auth-service/backend/pom.xml` with Spring Security, Jakarta Validation, Nimbus JWT (`nimbus-jose-jwt:9.37.3`), Testcontainers, and JaCoCo plugin (≥80%)
- [x] Create `User` domain model and domain exceptions (`DuplicateUserException`, `InvalidCredentialsException`, `UserValidationException`) with zero framework imports
- [x] Create inbound ports (`RegisterUserUseCase`, `LoginUserUseCase`), commands, and outbound ports (`UserRepositoryPort`, `PasswordEncoderPort`, `TokenProviderPort`)
- [x] Implement `AuthService` application service for registration and login
- [x] Implement `UserPersistenceAdapter`, `BCryptPasswordEncoderAdapter`, and `JwtTokenProviderAdapter` (using `${JWT_SECRET}`)
- [x] Implement `AuthController` (`POST /api/auth/register`, `POST /api/auth/login`), request/response DTOs, and RFC 7807 `GlobalExceptionHandler`
- [x] Configure stateless `SecurityConfig` in `auth-service`
- [x] Connect frontend `/login` and `/register` to `auth-service`, persist JWT, and redirect to `/characters` ("Em Construção" com link para protótipo público)

## Security Validation Tasks

- [x] Verify `@Valid` Jakarta Validation annotations on `RegisterRequest` and `LoginRequest`
- [x] Confirm `JWT_SECRET` is injected exclusively via environment variables (`${JWT_SECRET}`) with ≥ 256-bit key validation
- [x] Verify raw passwords and password hashes are never exposed in responses or logs

## Shift-Left Test Suite Tasks

### Happy Path
- [x] Write unit and MockMvc tests for `POST /api/auth/register` returning `201 Created` with JWT (TP-01)
- [x] Write unit and MockMvc tests for `POST /api/auth/login` returning `200 OK` with JWT (TP-02)
- [x] Write Testcontainers integration test for `UserPersistenceAdapter` (TP-03)

### Edge Cases & Failure Modes
- [x] Write unit and MockMvc tests for duplicate email/username returning `409 Conflict` ProblemDetail (TE-01)
- [x] Write unit and MockMvc tests for invalid credentials returning `401 Unauthorized` ProblemDetail (TE-02)
- [x] Write MockMvc test for invalid DTO payload returning `422 Unprocessable Entity` ProblemDetail (TE-03)

## Documentation Tasks

- [x] Update `specs/system-architecture-delta.md` with `auth-service` endpoints and JWT security flow
- [x] Update `walkthrough.md` with cumulative record of all phases

## Review Gate

- [x] All implementation tasks complete
- [x] All security validation tasks complete
- [x] All tests passing (`mvn -f auth-service/backend/pom.xml test verify -DskipTests`)
- [x] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [x] Ready for `sdd-spec-archive`
