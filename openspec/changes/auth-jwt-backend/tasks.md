# Tasks: Auth Jwt Backend

> **Change ID:** `auth-jwt-backend`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [ ] Configure `auth-service/backend/pom.xml` with Spring Security, Jakarta Validation, Nimbus JWT (`spring-boot-starter-oauth2-resource-server`), Testcontainers, and JaCoCo plugin (≥80%)
- [ ] Create `User` domain model and domain exceptions (`DuplicateUserException`, `InvalidCredentialsException`) with zero framework imports
- [ ] Create inbound ports (`RegisterUserUseCase`, `LoginUserUseCase`), commands, and outbound ports (`UserRepositoryPort`, `PasswordEncoderPort`, `TokenProviderPort`)
- [ ] Implement `AuthService` application service for registration and login
- [ ] Implement `UserPersistenceAdapter`, `BCryptPasswordEncoderAdapter`, and `JwtTokenProviderAdapter` (using `${JWT_SECRET}`)
- [ ] Implement `AuthController` (`POST /api/auth/register`, `POST /api/auth/login`), request/response DTOs, and RFC 7807 `GlobalExceptionHandler`
- [ ] Configure stateless `SecurityConfig` in `auth-service`

## Security Validation Tasks

- [ ] Verify `@Valid` Jakarta Validation annotations on `RegisterRequest` and `LoginRequest`
- [ ] Confirm `JWT_SECRET` is injected exclusively via environment variables (`${JWT_SECRET}`) with ≥ 256-bit key validation
- [ ] Verify raw passwords and password hashes are never exposed in responses or logs

## Shift-Left Test Suite Tasks

### Happy Path
- [ ] Write unit and MockMvc tests for `POST /api/auth/register` returning `201 Created` with JWT (TP-01)
- [ ] Write unit and MockMvc tests for `POST /api/auth/login` returning `200 OK` with JWT (TP-02)
- [ ] Write Testcontainers integration test for `UserPersistenceAdapter` (TP-03)

### Edge Cases & Failure Modes
- [ ] Write unit and MockMvc tests for duplicate email/username returning `409 Conflict` ProblemDetail (TE-01)
- [ ] Write unit and MockMvc tests for invalid credentials returning `401 Unauthorized` ProblemDetail (TE-02)
- [ ] Write MockMvc test for invalid DTO payload returning `422 Unprocessable Entity` ProblemDetail (TE-03)

## Documentation Tasks

- [ ] Update `specs/system-architecture-delta.md` with `auth-service` endpoints and JWT security flow

## Review Gate

- [ ] All implementation tasks complete
- [ ] All security validation tasks complete
- [ ] All tests passing (`mvn -f auth-service/backend/pom.xml clean verify`)
- [ ] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [ ] Ready for `sdd-spec-archive`
