Focus: Full-Stack / Spring Boot 4 (Java 21 / Clean Architecture) & Next.js 16 (React 19 / TypeScript / TailwindCSS 4)
Task: Execute Full Phased Implementation of Change `add-user-role-management`

## Execution Context
- Active Plan: `openspec/changes/add-user-role-management/tasks.md`
- Linked Proposal: `openspec/changes/add-user-role-management/proposal.md`
- Linked Delta Spec: `openspec/changes/add-user-role-management/specs/system-architecture-delta.md`
- Core Objectives:
  1. Fix the registration role persistence in `auth-service` so users selecting both Player and Dungeon Master modules are persisted with both `PLAYER` and `DUNGEON_MASTER` roles, falling back to `[PLAYER]` if no roles are explicitly provided.
  2. Support flexible birthdate parsing in `auth-service` allowing the standard Brazilian date format `dd/MM/yyyy` alongside ISO `yyyy-MM-dd`.
  3. Implement a self-promotion / role management endpoint (`POST /api/auth/me/roles` or `POST /api/auth/me/promote-dm`) in `auth-service` that adds the `DUNGEON_MASTER` role to the authenticated user and re-issues a fresh, signed JWT containing the updated role claims.
  4. Ensure `api-gateway` routes and propagates security headers (`X-User-Id`, `X-User-Roles`, `Authorization`) correctly for `/api/auth/me/**`.
  5. Update `RegisterForm` in `dungeoneer-frontend` to forward selected roles and provide an input mask and validation for `dd/MM/yyyy` birthdates.
  6. Create the user profile page (`app/profile/page.tsx`) allowing users to view account details, review active roles, and self-promote to `DUNGEON_MASTER`, seamlessly storing the new JWT token and triggering real-time UI permission updates.
  7. Add a direct navigation link to `/profile` in `components/navbar.tsx`.
  8. Deliver comprehensive automated tests (MockMvc integration tests on backend, Vitest component/unit tests on frontend).
- Architectural & Quality Constraints:
  - Clean Architecture (Hexagonal): Backend `domain/model/` must contain ZERO framework imports (no Spring, no JPA, no Lombok). Invariants must be enforced within domain entities.
  - Defensive Security: Input validation on roles and birthdate; `/api/auth/me/**` operations must act strictly on the authenticated user ID extracted from the verified token; secrets must remain isolated via environment variables.
  - Language Separation: Source code, test suites, and Git commit messages strictly in English. User-facing frontend strings, form labels, profile page text, and the shared walkthrough artifact strictly in Portuguese (`pt-BR`).
  - Build & Tools: Use global Maven (`mvn`), never the wrapper script (`mvnw`). Use global `npm`.
  - Phase Pauses & Conventional Commits: Execute sequentially through phases. Pause after each phase, update the cumulative walkthrough (`openspec/changes/add-user-role-management/walkthrough.md` in pt-BR), provide the exact ready-to-run Conventional Commit command, and await user approval before proceeding.

---

### Phase 1: Backend Domain Model & DTOs for Multi-Role Registration and Flexible Birthdate

#### Execution Context
- Scope: Execute Phase 1 only. Enhance the `User` domain model in `auth-service` to accept configurable roles on creation and add domain methods for role management. Update `RegisterRequest` and `RegisterUserCommand` to accept roles and parse `dd/MM/yyyy` birthdates.

#### Implementation Steps
1. Domain Layer (`auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java`):
   - Update `User.createNew` and constructors to accept a collection/set of `Role` objects.
   - If the provided roles collection is null or empty, default to a set containing `Role.PLAYER`.
   - Add domain method `addRole(Role role)` (or `promoteToDungeonMaster()`) that idempotently appends `Role.DUNGEON_MASTER` without removing existing roles.
   - Enforce domain invariants: roles collection cannot be empty, and birthdate must be in the past.
2. Web DTOs & Command Records (`auth-service/backend/`):
   - Update `RegisterRequest`:
     * Add `List<Role> roles` field (optional/nullable).
     * Configure flexible deserialization for `birthdate`: support `dd/MM/yyyy` (e.g. via `@JsonFormat(pattern = "dd/MM/yyyy")` or custom deserializer that handles both `dd/MM/yyyy` and ISO `yyyy-MM-dd`).
   - Update `RegisterUserCommand`:
     * Include `List<Role> roles` in the record definition.
3. Unit Testing:
   - Write unit tests in `UserTest.java` verifying:
     * Creation with multiple roles preserves all roles.
     * Creation with null or empty roles defaults to `PLAYER`.
     * `addRole` adds `DUNGEON_MASTER` while preserving existing `PLAYER` role.
   - Write unit tests in `RegisterRequestTest.java` (or DTO test) verifying date parsing for `dd/MM/yyyy` and `yyyy-MM-dd`.

#### Quality Gates & Verification
- Run backend unit tests using global Maven:
  ```bash
  mvn clean test -f auth-service/backend/pom.xml -Dtest=*UserTest*,*Register*
  ```
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/src/main/java/com/dungeoneer/auth/domain/ \
          auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/ \
          auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/dto/ \
          auth-service/backend/src/test/java/
  git commit -m "feat(auth): support multi-role domain creation and flexible birthdate parsing"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Initialize `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`):
  1. Summary of domain model updates and DTO deserialization enhancements.
  2. Unit test execution results.
  3. Pause and present the commit command to the user before proceeding to Phase 2.

---

### Phase 2: Backend Application Service, Self-Promotion Endpoint & JWT Re-issuance

#### Execution Context
- Scope: Execute Phase 2 only. Implement the self-promotion use case, register endpoint, and role update endpoint in `auth-service` that re-issues a signed JWT with updated claims.

#### Implementation Steps
1. Application Layer (`auth-service/backend/`):
   - Update `AuthService` registration flow:
     * Pass roles from `RegisterUserCommand` to `User.createNew`.
   - Implement self-promotion / role update logic in `AuthService`:
     * Create inbound port `SelfPromoteUseCase` (or add method to `AuthService`): `promoteToDungeonMaster(UUID userId)`.
     * Retrieve the user from `UserRepositoryPort`.
     * Invoke domain method to add `Role.DUNGEON_MASTER`.
     * Persist updated user through `UserRepositoryPort` / `UserPersistenceAdapter`.
     * Re-issue a newly minted JWT token containing the updated roles in its claims.
     * Return `AuthResponse` containing the new `accessToken`, token type, and user details.
2. Web Controller (`auth-service/backend/`):
   - Update `AuthController`:
     * Ensure `register` maps the incoming roles list into `RegisterUserCommand`.
     * Expose endpoint `POST /api/auth/me/roles` (or `POST /api/auth/me/promote-dm`):
       - Extract authenticated user ID from security context / auth principal.
       - Execute self-promotion.
       - Return `200 OK` with `AuthResponse`.
   - Return RFC 7807 `ProblemDetail` on errors (e.g. UserNotFoundException -> 404, Unauthorized -> 401).
3. Integration Tests:
   - Create or update MockMvc integration tests in `AuthControllerTest`:
     * Registering with `[PLAYER, DUNGEON_MASTER]` returns `201 Created` with both roles in the response and JWT claims.
     * Registering with `dd/MM/yyyy` birthdate succeeds.
     * Authenticated call to `POST /api/auth/me/roles` adds `DUNGEON_MASTER` and returns fresh token with updated claims.
     * Unauthenticated call to `/api/auth/me/roles` returns `401 Unauthorized`.

#### Quality Gates & Verification
- Run backend tests using global Maven:
  ```bash
  mvn clean test -f auth-service/backend/pom.xml
  ```
- Ready-to-use commit command:
  ```bash
  git add auth-service/backend/src/main/java/com/dungeoneer/auth/application/ \
          auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/ \
          auth-service/backend/src/test/java/
  git commit -m "feat(auth): implement self-promotion endpoint with JWT re-issuance"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2 section to `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`):
  1. Description of application use case, controller endpoints, and JWT claims renewal.
  2. MockMvc integration test results.
  3. Pause and present the commit command to the user before proceeding to Phase 3.

---

### Phase 3: API Gateway Security Configuration & Header Propagation

#### Execution Context
- Scope: Execute Phase 3 only. Verify that `api-gateway` routes `/api/auth/me/**` to `auth-service` and properly validates and propagates user authentication headers.

#### Implementation Steps
1. Gateway Security Configuration (`api-gateway/backend/`):
   - Inspect route definitions and `JwtHeaderFilter`:
     * Verify that `/api/auth/me/**` requires authentication (unlike public `/api/auth/login` and `/api/auth/register`).
     * Ensure the gateway verifies the incoming Bearer token and forwards user identity headers (`X-User-Id`, `X-User-Roles`, `Authorization`) down to `auth-service`.
     * Verify that CORS allows `POST` and `PUT` methods with `Authorization` headers.
2. Integration & Unit Tests:
   - Run existing or create test cases in `api-gateway` verifying route protection:
     * Request to `/api/auth/me/roles` without token returns `401 Unauthorized`.
     * Request with valid JWT routes to downstream with proper headers.

#### Quality Gates & Verification
- Run gateway tests using global Maven:
  ```bash
  mvn clean test -f api-gateway/backend/pom.xml
  ```
- Ready-to-use commit command:
  ```bash
  git add api-gateway/backend/
  git commit -m "feat(gateway): configure auth routing and header propagation for /api/auth/me/**"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 3 section to `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`):
  1. Gateway routing rules and security filter configurations verified.
  2. Test execution outputs.
  3. Pause and present the commit command to the user before proceeding to Phase 4.

---

### Phase 4: Frontend Registration Form & Birthdate Formatting

#### Execution Context
- Scope: Execute Phase 4 only. Update the frontend registration client and form component to send multiple roles and handle `dd/MM/yyyy` birthdates with an interactive mask and validation.

#### Implementation Steps
1. API Client (`dungeoneer-frontend/lib/api/auth.ts`):
   - Update `RegisterData` interface to include optional `roles?: ('PLAYER' | 'DUNGEON_MASTER')[]`.
   - Ensure `register` function transmits the `roles` array when provided.
2. Register Form Component (`dungeoneer-frontend/components/register-form.tsx`):
   - Checkbox Mapping:
     * Map module selection options (Player / Dungeon Master) directly to the `roles` payload array (e.g. if both are checked, send `['PLAYER', 'DUNGEON_MASTER']`).
   - Birthdate Mask & Validation:
     * Add input masking for `dd/MM/yyyy` (auto-inserting slashes as the user types).
     * Validate date format and ensure the entered birthdate represents a valid calendar day in the past.
     * Display clear, localized pt-BR validation messages (e.g. "Data de nascimento inválida", "A data deve estar no passado").
3. Frontend Unit Tests:
   - Create or update tests in `dungeoneer-frontend/__tests__/register-form.test.tsx`:
     * Test selecting both Player and DM modules includes both roles in the submission payload.
     * Test typing in the birthdate input applies the `dd/MM/yyyy` format mask.
     * Test submitting an invalid date displays Portuguese validation error.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/lib/api/auth.ts \
          dungeoneer-frontend/components/register-form.tsx \
          dungeoneer-frontend/__tests__/
  git commit -m "feat(frontend): support multiple roles and dd/MM/yyyy birthdate in RegisterForm"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 4 section to `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`):
  1. Form enhancements, role mapping, and input mask behavior.
  2. Unit test verification results.
  3. Pause and present the commit command to the user before proceeding to Phase 5.

---

### Phase 5: Frontend Profile Page & Navigation Link

#### Execution Context
- Scope: Execute Phase 5 only. Build the `/profile` page for viewing user details and promoting the current user to Dungeon Master, and link it from the main `Navbar`.

#### Implementation Steps
1. API & Auth Helpers (`dungeoneer-frontend/lib/api/auth.ts`):
   - Implement `selfPromoteToDm()` API function calling `POST /api/auth/me/roles`.
   - On successful response:
     * Store the newly issued `accessToken` in localStorage / cookies.
     * Update stored user profile data.
     * Dispatch `window.dispatchEvent(new Event('auth-state-change'))` to notify all reactive listeners (`useAuth()`, `Navbar`, etc.).
2. User Profile Page (`dungeoneer-frontend/app/profile/page.tsx`):
   - Dark Fantasy RPG card layout displaying:
     * User avatar/initials, nickname, email, birthdate (formatted in pt-BR).
     * Active role badges (`Jogador`, `Mestre do Jogo`).
   - DM Self-Promotion Card:
     * If user is NOT a Dungeon Master:
       - Show informative card detailing DM capabilities (Campaign management, NPC/Monster tools, Homebrew content).
       - Provide CTA button: "Tornar-se Mestre de Jogo (DM)".
       - On click, execute `selfPromoteToDm()`, show success toast/alert ("Agora você é um Mestre de Jogo!"), and update active badges dynamically without requiring a logout/login cycle.
     * If user IS ALREADY a Dungeon Master:
       - Display verified badge: "Mestre de Jogo Ativo" with descriptive text confirming full access to DM features.
3. Navbar Integration (`dungeoneer-frontend/components/navbar.tsx`):
   - Add "Meu Perfil" link to the user dropdown or main navigation bar navigating to `/profile`.
4. Unit & Component Tests:
   - Create `dungeoneer-frontend/__tests__/profile-page.test.tsx`:
     * Verify rendering of user information and existing roles.
     * Verify self-promotion action triggers API call and displays success message.
     * Verify DM badge displays when user already has `DUNGEON_MASTER` role.

#### Quality Gates & Verification
- Run frontend unit tests and production build:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/profile/ \
          dungeoneer-frontend/components/navbar.tsx \
          dungeoneer-frontend/lib/api/auth.ts \
          dungeoneer-frontend/__tests__/
  git commit -m "feat(frontend): create profile page with DM self-promotion and update navbar"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 5 section to `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`):
  1. Profile page UI components, self-promotion workflow, and state updates.
  2. Component test logs and build verification.
  3. Pause and present the commit command to the user before proceeding to Phase 6.

---

### Phase 6: Full-Stack Verification, Documentation & Final Walkthrough

#### Execution Context
- Scope: Execute Phase 6 only. Run complete regression tests across backend and frontend, update the delta specification, mark all tasks complete in `tasks.md`, and generate the final cumulative walkthrough.

#### Implementation Steps
1. Full-Stack Quality Verification:
   - Execute backend test suites:
     ```bash
     mvn clean test -f auth-service/backend/pom.xml
     mvn clean test -f api-gateway/backend/pom.xml
     ```
   - Execute frontend unit tests and production build:
     ```bash
     npm --prefix dungeoneer-frontend run test:unit
     npm --prefix dungeoneer-frontend run build
     ```
2. Specification & Task Checklist:
   - Update delta spec `openspec/changes/add-user-role-management/specs/system-architecture-delta.md`:
     * Document the `/api/auth/me/roles` endpoint specification and request/response schema.
     * Document multi-role registration behavior and birthdate format support.
     * Document JWT token refresh mechanism on role changes.
   - Mark all tasks in `openspec/changes/add-user-role-management/tasks.md` as completed (`[x]`).
3. Finalize Cumulative Walkthrough:
   - Consolidate all execution notes into `openspec/changes/add-user-role-management/walkthrough.md` in Portuguese (`pt-BR`).
   - Include a clear manual validation checklist:
     * Registering a new account with both Player and DM options and inspecting the decoded JWT roles.
     * Registering with birthdate `dd/MM/yyyy`.
     * Logging in as a Player-only account, visiting `/profile`, clicking "Tornar-se Mestre de Jogo (DM)", and verifying instant unlocking of DM categories without re-logging.
   - Confirm readiness for archiving via `sdd-spec-archive`.

#### Quality Gates & Verification
- Verify all checks pass across the repository.
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/add-user-role-management/tasks.md \
          openspec/changes/add-user-role-management/specs/system-architecture-delta.md \
          openspec/changes/add-user-role-management/walkthrough.md
  git commit -m "docs(spec): finalize tasks and cumulative walkthrough for add-user-role-management"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Deliver final summary and indicate readiness for archival via `sdd-spec-archive`.
