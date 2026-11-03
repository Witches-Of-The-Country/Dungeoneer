# Phased Execution Prompt: Fix HTTP 500 Errors and Enhance RFC 7807 Diagnostics

Focus: Full-Stack / Spring Boot 4 (Java 21) / PostgreSQL 15 JSONB / Clean Architecture / Next.js 16 (React 19 / TypeScript)
Task: Execute Full Phased Implementation of Change `001-fix-500-errors-and-rfc7807-diagnostics`

## Global Execution Directives
- **Active Change Plan:** `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- **Approved Proposal:** `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/proposal.md`
- **Delta Spec:** `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/specs/player-character-domain-delta.md`
- **Cumulative Walkthrough Artifact:** `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` (maintained strictly in Portuguese `pt-BR`).
- **Architectural & Stack Invariants:**
  - **Clean Architecture (Hexagonal):** Zero framework imports in backend `domain/model/` (no Spring, JPA, Hibernate, or Jackson annotations). Web serialization annotations and boundary DTOs must reside strictly within `application/dto/` and `adapter/in/web/`. JSONB persistence mappings reside strictly within `adapter/out/persistence/`.
  - **Build & Test Tools:** ALWAYS use global `mvn` (e.g., `mvn -f player-character-service/backend/pom.xml ...`), NEVER maven wrapper (`mvnw`). Maintain line coverage `>= 80%` on `domain` and `application` packages.
  - **Error Handling:** Return RFC 7807 `ProblemDetail` responses for all REST errors via `@ControllerAdvice`.
  - **Phase Pauses & Commit Commands:** Execute one phase at a time. After verifying each phase locally, provide the exact Conventional Commit command, update the cumulative walkthrough in `pt-BR`, mark completed tasks as `[x]` in `tasks.md`, and pause for user confirmation before proceeding to the next phase.

---

### Phase 1: Character Feat JSONB Serialization & Cascade Persistence

Focus: Backend / Java 21 / Spring Boot / PostgreSQL JSONB Persistence
Task: Execute Phase 1: Fix Character Feat JSONB Serialization and Deduplicate Initial Feats

#### Execution Context
- Active Plan: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- Proposal Reference: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/proposal.md`
- Scope: Execute Phase 1 only. Resolve the PostgreSQL `invalid input syntax for type json` and unique constraint violations during character creation with feats. Do not proceed to homebrew item or frontend changes.

#### Implementation Steps
1. **Sanitize `CharacterFeat` Instantiation:**
   - In `CreateCharacterService` and `LevelUpService`, ensure the `choicesJson` constructor parameter is always initialized with a valid JSON string (e.g., `"{}"` or `{"source":"..."}`) instead of plain enum names or unquoted tokens.
2. **Deduplicate Feats & Fix Persistence Cascade:**
   - In `CreateCharacterService`, deduplicate initial feats by `featId` to avoid inserting duplicate feats when background and class progression overlap.
   - Rely on cascade persistence from `PlayerCharacter` or ensure single-pass insertion without redundant subsequent loops over `saveCharacterFeat(cf)`.
3. **Defensive Normalization in Persistence Layer:**
   - In `CharacterFeatJpaEntity.fromDomain()`, ensure any input string is defensively validated and serialized as valid JSON before binding to the PostgreSQL `JSONB` column.
4. **Unit Tests:**
   - Add/update unit tests in `CreateCharacterServiceTest` verifying that character creation with Level > 1 and Origin Feats produces valid JSON choices and avoids duplicate persistence calls.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=CreateCharacterServiceTest
  ```
- Commit locally:
  ```bash
  git commit -m "fix(character): sanitize feat choices jsonb and prevent duplicate cascade persistence"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update or create `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` in pt-BR recording:
1. List of files modified in Phase 1.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 2.

---

### Phase 2: Homebrew Item Mechanics Boundary DTO & JSONB Discriminator Round-Trip

Focus: Backend / Clean Architecture Web Boundary / Jackson Serialization / PostgreSQL JSONB
Task: Execute Phase 2: Boundary DTO for Homebrew Item Mechanics & Lossless JSONB Round-Trip

#### Execution Context
- Active Plan: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- Proposal Reference: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/proposal.md`
- Scope: Execute Phase 2 only. Fix Jackson `InvalidDefinitionException` when creating homebrew items with mechanical properties, preserving clean domain interfaces without framework annotations.

#### Implementation Steps
1. **Web Boundary DTO & Web Mapping:**
   - Introduce a structured boundary DTO (e.g., `HomebrewItemMechanicsRequest` in `application/dto/`) to receive `mechanicalProperties` from the incoming JSON payload in `CreateHomebrewItemRequest`.
   - In `ReferenceDataWebMapper`, cleanly map this boundary DTO into the sealed domain hierarchy (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, or `EmptyItemMechanics`) without adding any Jackson annotations to domain model classes.
2. **JSONB Discriminator & Lossless Persistence:**
   - In `ItemJpaEntity.serializeMechanics()`, ensure the JSONB serialization includes the `"type"` discriminator (`"WEAPON"`, `"ARMOR"`, `"CONSUMABLE"`).
   - In `ItemJpaEntity.deserializeMechanics()`, verify that mechanical properties reconstruct domain records accurately with defensive fallback for existing records lacking `"type"`.
3. **Unit Tests:**
   - Add/update unit tests in `HomebrewServiceTest` and `ReferenceDataControllerTest` validating successful deserialization, domain mapping, and JSONB persistence round-trips for weapons, armor, and consumables.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=HomebrewServiceTest,ReferenceDataControllerTest
  ```
- Commit locally:
  ```bash
  git commit -m "fix(reference): isolate item mechanics boundary dto and ensure jsonb type discriminator round-trip"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 2 progress to `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` in pt-BR:
1. List of files created/modified in Phase 2.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 3.

---

### Phase 3: Backend RFC 7807 Exception Handling & Diagnostics

Focus: Backend / Spring Web / RFC 7807 ProblemDetail / Global Diagnostics
Task: Execute Phase 3: Implement RFC 7807 ProblemDetail Handlers for Data Integrity & Fallback Errors

#### Execution Context
- Active Plan: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- Proposal Reference: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/proposal.md`
- Scope: Execute Phase 3 only. Enhance backend error responses so unexpected database errors and unhandled exceptions return structured, actionable RFC 7807 `ProblemDetail` payloads instead of opaque 500 error responses.

#### Implementation Steps
1. **Add Handlers in `GlobalExceptionHandler`:**
   - Add `@ExceptionHandler(DataIntegrityViolationException.class)` returning HTTP 409 Conflict with sanitized, diagnostic pt-BR detail explaining the constraint or conflict without leaking sensitive credentials or raw queries.
   - Add fallback `@ExceptionHandler(Exception.class)` returning HTTP 500 Internal Server Error returning an RFC 7807 `ProblemDetail` containing `title`, `detail`, `status`, and `path`.
2. **Unit Tests:**
   - Add tests in `GlobalExceptionHandlerTest` verifying that `DataIntegrityViolationException` and unhandled runtime exceptions produce compliant RFC 7807 responses with proper HTTP status, title, and detail attributes.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=GlobalExceptionHandlerTest
  ```
- Commit locally:
  ```bash
  git commit -m "fix(core): add rfc 7807 problem detail handlers for data integrity and generic exceptions"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 3 progress to `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` in pt-BR:
1. List of files modified in Phase 3.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 4.

---

### Phase 4: Frontend RFC 7807 Diagnostic Display & Toast Formatting

Focus: Frontend / Next.js 16 / TypeScript / Error Toast Presentation
Task: Execute Phase 4: Enhance API Client RFC 7807 Extraction & User Diagnostics

#### Execution Context
- Active Plan: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- Proposal Reference: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/proposal.md`
- Scope: Execute Phase 4 only. Update frontend API client error parsing to extract RFC 7807 `detail` and `title` so users receive informative toast error notifications in pt-BR.

#### Implementation Steps
1. **Enhance `lib/api/client.ts`:**
   - When handling non-2xx responses in `apiFetch`, parse the response body as RFC 7807 `ProblemDetail`.
   - Prefer `detail`, falling back to `title`, `message`, `error`, or formatted HTTP status + path.
   - Ensure `ApiError` exposes this structured diagnostic message so frontend toast notifications and mutation error handlers display human-readable Portuguese messages.
2. **Unit Tests:**
   - Add/update unit tests in `dungeoneer-frontend/lib/api/client.test.ts` verifying parsing of RFC 7807 error responses, fallback strings, and error extraction.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "fix(frontend): extract and display rfc 7807 problem details in api client"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 4 progress to `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` in pt-BR:
1. List of files modified in Phase 4.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 5.

---

### Phase 5: Verification, Spec Archival Preparation & Final Walkthrough

Focus: Full-Stack Quality Assurance / SDD Spec Finalization
Task: Execute Phase 5: Run Full Test Suites & Finalize Walkthrough

#### Execution Context
- Active Plan: `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md`
- Scope: Execute Phase 5 only. Verify full regression test suites, ensure all tasks in `tasks.md` are marked `[x]`, and finalize the cumulative walkthrough artifact.

#### Implementation Steps
1. **Full Verification:**
   - Run complete test suite for `player-character-service` backend and `dungeoneer-frontend`.
   - Verify JaCoCo line coverage meets `>= 80%` on domain and application layers.
2. **Tasks & Delta Spec Finalization:**
   - Mark all tasks in `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/tasks.md` as `[x]`.
   - Verify `specs/player-character-domain-delta.md` accurately documents the changes.
3. **Consolidated Walkthrough Summary:**
   - Finalize `openspec/changes/001-fix-500-errors-and-rfc7807-diagnostics/walkthrough.md` with:
     - Consolidated summary of all changes across Phases 1–4.
     - Full test results and coverage metrics.
     - Manual verification checklist for smoke testing character creation and homebrew item creation.
     - Readiness confirmation for archiving via `sdd-spec-archive`.

#### Quality Gates & Verification
- Run validation commands:
  ```bash
  mvn -f player-character-service/backend/pom.xml clean test
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "docs(spec): mark change 001 tasks complete and record final walkthrough"
  ```
