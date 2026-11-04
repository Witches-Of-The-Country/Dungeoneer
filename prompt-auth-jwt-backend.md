Focus: Fullstack / Spring Boot 4 / Java 21 / Next.js 16 / React 19 / Clean Architecture
Task: Execute Implementation of Change `auth-jwt-backend` & Frontend Auth Integration with Under Construction / Prototype Route

## Execution Context
- Active Plan: `openspec/changes/auth-jwt-backend/tasks.md`
- Linked Proposal: `openspec/changes/auth-jwt-backend/proposal.md`
- Delta Spec: `openspec/changes/auth-jwt-backend/specs/system-architecture-delta.md`
- Target Areas:
  - Backend: `auth-service/backend`
  - Frontend: `dungeoneer-frontend`
- Quality & Architecture Contract:
  - Strict Clean Architecture (Hexagonal): domain model MUST have zero framework imports (no Spring, JPA, or Lombok).
  - TDD workflow (Red -> Green -> Refactor): cover happy paths and edge cases simultaneously.
  - Secret isolation: inject `${JWT_SECRET}` strictly via environment variables; fail fast on startup if key is < 256 bits (32 bytes). Never leak raw passwords or hashes.
  - Global Maven: always invoke `mvn` (e.g., `mvn -f auth-service/backend/pom.xml ...`), never wrapper.
  - Coverage Gate: JaCoCo line coverage must be ≥ 80% on domain and application layers.
  - Frontend Integration: connect existing login and registration forms to the auth backend. Upon successful registration or login, persist the JWT in client storage and redirect the user to an "Em Construção" (Under Construction) character sheet view that links to or renders the character creation prototype (`/prototype/character-creation`), which remains publicly accessible.
  - Phase Pauses & Commits: execute one phase at a time, verify locally, present the ready-to-use commit command, update the cumulative walkthrough (`openspec/changes/auth-jwt-backend/walkthrough.md` in pt-BR), and pause for user approval before moving to the next phase.

---

### Phase 1: Dependencies, Build Setup & Configuration

#### Execution Context
- Scope: Execute Phase 1 only. Do not proceed to subsequent phases until verified and committed.

#### Implementation Steps
1. Configure `auth-service/backend/pom.xml`:
   - Add `spring-boot-starter-security`.
   - Add `spring-boot-starter-validation`.
   - Add `spring-boot-starter-oauth2-resource-server` (or Nimbus JOSE + JWT).
   - Add `testcontainers` and PostgreSQL testcontainer dependencies.
   - Configure `jacoco-maven-plugin` enforcing minimum 80% line coverage for domain and application packages.
2. Configure `auth-service/backend/src/main/resources/application.properties`:
   - Add `jwt.secret=${JWT_SECRET}` and `jwt.expiration-seconds=${JWT_EXPIRATION_SECONDS:86400}`.
3. Configure `AuthServiceApplication`:
   - Update component scanning with `@SpringBootApplication(scanBasePackages = "com.dungeoneer.auth")` to guarantee cross-package discovery.

#### Quality Gates & Verification
- Compile project and resolve dependencies:
  `mvn -f auth-service/backend/pom.xml clean compile`
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/pom.xml auth-service/backend/src/main/resources/application.properties auth-service/backend/src/main/java/com/dungeoneer/auth/AuthServiceApplication.java
  git commit -m "build(auth): configure security, validation, jwt and test dependencies"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Create `openspec/changes/auth-jwt-backend/walkthrough.md` in Portuguese (pt-BR) recording:
  1. Files created/modified during Phase 1.
  2. Compilation and dependency verification status.
  3. Pause and present the commit command to the user before advancing to Phase 2.

---

### Phase 2: Domain Layer & Unit Tests (TDD)

#### Execution Context
- Scope: Execute Phase 2 only. Maintain zero framework imports in the domain package.

#### Implementation Steps
1. Implement pure Java `User` domain model in `com.dungeoneer.auth.domain.model.User`:
   - Model invariants: UUID id, email, nickname, birthdate, passwordHash, and roles.
   - Business invariants: enforce non-null fields, valid email format, birthdate in the past.
   - Zero framework imports: no Spring, no JPA, no Lombok.
2. Implement domain exceptions in `com.dungeoneer.auth.domain.exception`:
   - `DuplicateUserException` (for email or username collisions).
   - `InvalidCredentialsException` (for authentication failures).
3. Write domain unit tests:
   - Validate invariant enforcement on valid/invalid inputs.
   - Verify role assignment and immutability.

#### Quality Gates & Verification
- Run domain unit tests:
  `mvn -f auth-service/backend/pom.xml test -Dtest="com.dungeoneer.auth.domain.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/src/main/java/com/dungeoneer/auth/domain/ auth-service/backend/src/test/java/com/dungeoneer/auth/domain/
  git commit -m "feat(auth): implement User domain model, exceptions and domain unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/auth-jwt-backend/walkthrough.md` (pt-BR):
  1. Domain models and exceptions created.
  2. Domain unit test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 3.

---

### Phase 3: Application Layer (Ports, Use Cases, DTOs & Services)

#### Execution Context
- Scope: Execute Phase 3 only.

#### Implementation Steps
1. Define inbound ports in `com.dungeoneer.auth.application.port.in`:
   - `RegisterUserUseCase`
   - `LoginUserUseCase`
2. Define outbound ports in `com.dungeoneer.auth.application.port.out`:
   - `UserRepositoryPort` (`save`, `findByEmail`, `findByNickname`, `existsByEmail`, `existsByNickname`)
   - `PasswordEncoderPort` (`encode`, `matches`)
   - `TokenProviderPort` (`generateToken`)
3. Define application DTOs and commands in `com.dungeoneer.auth.application.dto`:
   - `RegisterUserCommand`, `LoginUserCommand`, and `AuthResult` (token, tokenType, expiresIn, user summary).
4. Implement `AuthService` in `com.dungeoneer.auth.application.service`:
   - Registration flow: normalize email, verify uniqueness via `UserRepositoryPort`, hash password via `PasswordEncoderPort`, persist `User`, and generate signed JWT.
   - Login flow: lookup user by email, verify password hash via `PasswordEncoderPort`, issue JWT with `sub = user.getId()`. Throw `InvalidCredentialsException` on unknown email or password mismatch (avoid user enumeration).
5. Write Mockito unit tests for `AuthService`:
   - Happy paths: user registration (TP-01 logic) and user login (TP-02 logic).
   - Failure modes: duplicate email/username (TE-01 logic) and invalid credentials (TE-02 logic).

#### Quality Gates & Verification
- Run application layer unit tests:
  `mvn -f auth-service/backend/pom.xml test -Dtest="com.dungeoneer.auth.application.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/src/main/java/com/dungeoneer/auth/application/ auth-service/backend/src/test/java/com/dungeoneer/auth/application/
  git commit -m "feat(auth): implement application ports, use cases, auth service and unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/auth-jwt-backend/walkthrough.md` (pt-BR):
  1. Ports, commands, and service implementations created.
  2. Application unit test results.
  3. Pause and present the commit command to the user before advancing to Phase 4.

---

### Phase 4: Adapters (Persistence, Security, Web) & Integration Tests

#### Execution Context
- Scope: Execute Phase 4 only.

#### Implementation Steps
1. Persistence Adapter:
   - Align `UserJpaEntity` with `V1__create_users_table.sql` and domain mapping.
   - Implement `UserPersistenceAdapter` implementing `UserRepositoryPort`.
   - Write Testcontainers integration test with PostgreSQL 15 verifying `UserPersistenceAdapter` persistence and queries (TP-03).
2. Security Adapters:
   - Implement `BCryptPasswordEncoderAdapter` implementing `PasswordEncoderPort`.
   - Implement `JwtTokenProviderAdapter` implementing `TokenProviderPort` using HMAC-SHA256 (`${JWT_SECRET}`). Fail fast on initialization if `JWT_SECRET` is missing or < 32 bytes (256 bits).
   - Configure stateless `SecurityConfig`: permit `POST /api/auth/**` and `/actuator/health`, disable CSRF, set session creation policy to `STATELESS`.
3. Web Adapter:
   - Implement DTOs with Jakarta Validation: `RegisterRequest` (`@NotBlank @Email email`, `@NotBlank @Size(min = 3, max = 50) nickname`, `@NotNull birthdate`, `@NotBlank @Size(min = 8, max = 100) password`), `LoginRequest` (`@NotBlank @Email email`, `@NotBlank password`), and `AuthResponse`.
   - Implement `AuthController` with endpoints:
     - `POST /api/auth/register` returning `201 Created` with `AuthResponse`.
     - `POST /api/auth/login` returning `200 OK` with `AuthResponse`.
   - Implement `GlobalExceptionHandler` returning RFC 7807 `ProblemDetail` for:
     - `DuplicateUserException` -> `409 Conflict`
     - `InvalidCredentialsException` -> `401 Unauthorized`
     - `MethodArgumentNotValidException` -> `422 Unprocessable Entity`
4. Web API Tests (MockMvc):
   - TP-01: Valid registration returns `201 Created` with signed JWT and user info.
   - TP-02: Valid login returns `200 OK` with signed JWT.
   - TE-01: Duplicate user returns `409 Conflict` ProblemDetail.
   - TE-02: Bad login returns `401 Unauthorized` ProblemDetail.
   - TE-03: Invalid DTO payload returns `422 Unprocessable Entity` ProblemDetail.

#### Quality Gates & Verification
- Run all test suites:
  `mvn -f auth-service/backend/pom.xml test`
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/ auth-service/backend/src/main/java/com/dungeoneer/auth/core/ auth-service/backend/src/test/java/com/dungeoneer/auth/adapter/
  git commit -m "feat(auth): implement persistence, security adapters, REST controller and web tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/auth-jwt-backend/walkthrough.md` (pt-BR):
  1. Adapters, security configurations, and REST endpoints created.
  2. MockMvc and Testcontainers test results.
  3. Pause and present the commit command to the user before advancing to Phase 5.

---

### Phase 5: Frontend Auth Integration & "Em Construção" Destination Route

#### Execution Context
- Scope: Execute Phase 5. Connect `dungeoneer-frontend` to `auth-service` via API calls, persist JWT, redirect upon login/register, and establish the "Em Construção" view with public prototype access.

#### Implementation Steps
1. Configure frontend environment and API client helper:
   - Provide client helper targeting auth endpoints (`NEXT_PUBLIC_AUTH_API_URL` or fallback `http://localhost:8082` for direct access until gateway routing phase).
2. Wire authentication in `components/login-form.tsx` and `components/register-form.tsx`:
   - Send `POST /api/auth/login` and `POST /api/auth/register`.
   - On success: persist JWT token and user info in `localStorage`.
   - Trigger toast notification on error parsing RFC 7807 `ProblemDetail`.
   - Redirect user on success to `/characters`.
3. Create "Em Construção" (Under Construction) character page in `app/characters/page.tsx`:
   - Displays a Dark Fantasy themed "Em Construção" message for the character sheet list.
   - Includes a prominent action button allowing the user to test the wizard prototype (`/prototype/character-creation`).
4. Ensure public prototype route remains accessible:
   - Verify `/prototype/character-creation` renders without requiring mandatory authentication blockers.

#### Quality Gates & Verification
- Test build and linting of frontend:
  `npm --prefix dungeoneer-frontend run lint`
  `npm --prefix dungeoneer-frontend run build`
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/
  git commit -m "feat(frontend): connect login and register to auth backend with under-construction redirect"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/auth-jwt-backend/walkthrough.md` (pt-BR):
  1. Frontend components updated with live API integration.
  2. "Em Construção" page and prototype linking.
  3. Pause and present the commit command to the user before advancing to Phase 6.

---

### Phase 6: Verification, Architecture Delta & Final Consolidation

#### Execution Context
- Scope: Execute Phase 6. Finalize change validation and cumulative documentation.

#### Implementation Steps
1. Run full build and verify JaCoCo coverage on backend:
   - Ensure JaCoCo line coverage is ≥ 80% on domain and application packages.
2. Update specification delta in `openspec/changes/auth-jwt-backend/specs/system-architecture-delta.md`:
   - Document new endpoints (`POST /api/auth/register`, `POST /api/auth/login`), JWT signing algorithm, security posture, and frontend auth wiring.
3. Update `openspec/changes/auth-jwt-backend/tasks.md`:
   - Mark all completed tasks with `[x]`.
4. Finalize `openspec/changes/auth-jwt-backend/walkthrough.md` in Portuguese (pt-BR):
   - Consolidate all changes across Phases 1 through 6.
   - Document test verification and coverage summary.
   - Provide clear actionable next steps.

#### Quality Gates & Verification
- Run full Maven build and verification:
  `mvn -f auth-service/backend/pom.xml clean verify`
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/auth-jwt-backend/ prompt-auth-jwt-backend.md
  git commit -m "chore(auth): complete verification, architecture delta and walkthrough documentation"
  ```

#### Final Phase Handoff
- Present the consolidated walkthrough to the user and notify readiness for `sdd-spec-validate` or `sdd-spec-archive`.
