# Tasks: Add Homebrew Creation

> **Change ID:** `add-homebrew-creation`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### 1. Database & Persistence Layer
- [x] Create Flyway migration `V15__enable_homebrew_ruleset.sql` in `player-character-service` adding `HOMEBREW` to CHECK constraints on `archetypes`, `lineages`, and `backgrounds`
- [x] Add composite indexes `(ruleset, owner_id)` across all reference tables in `V15__enable_homebrew_ruleset.sql`
- [x] Update JPA reference entities with `@Column(name = "owner_id") private UUID ownerId;`
- [x] Update `ReferenceDataPersistenceAdapter` query implementations to support visibility filter `(owner_id IS NULL OR owner_id = :userId)`

### 2. Domain & Application Layer
- [x] Update `Ruleset.java` in `player-character-service` domain layer to include `HOMEBREW`
- [x] Create inbound use case port `CreateHomebrewUseCase` in `application.port.in`
- [x] Implement request DTOs with Jakarta Validation in `application.dto`:
  - [x] `CreateHomebrewSpellRequest`
  - [x] `CreateHomebrewItemRequest`
  - [x] `CreateHomebrewFeatRequest`
  - [x] `CreateHomebrewLineageRequest`
  - [x] `CreateHomebrewArchetypeRequest`
- [x] Implement `HomebrewService` in `application.service` implementing `CreateHomebrewUseCase`

### 3. Web Layer & API Gateway
- [x] Implement `POST /api/reference/{category}` endpoints in `ReferenceDataController` returning HTTP 201 with `Location` header
- [x] Extract `X-User-Id` header in `ReferenceDataController` and pass creator ID to use cases and visibility queries
- [x] Update `JwtHeaderFilter` in `api-gateway` to require `DUNGEON_MASTER` for `POST /api/reference/**` routes
- [x] Configure `api-gateway` to strip incoming `X-User-Id` / `X-User-Roles` from external clients and propagate verified values from JWT claims

### 4. Frontend Integration
- [x] Update frontend types in `lib/api/characters.ts` to include `HOMEBREW` in `Ruleset` type
- [x] Create TanStack Query mutation hooks and API client methods in `lib/api/homebrew.ts`
- [x] Create `CreateHomebrewDialog` component with Zod/state validation and Portuguese (`pt-BR`) labels in `components/homebrew/create-homebrew-dialog.tsx`
- [x] Integrate "Criar novo" action button in `app/encyclopedia/{spells,items,feats,classes,lineages}/page.tsx` with DM role guard (`isDM`)
- [x] Add success toast notification and cache invalidation on mutation success

---

## Test Tasks

- [x] Unit tests for `HomebrewService` and domain logic using JUnit 5 + Mockito (`HomebrewServiceTest`)
- [x] Validation tests for `CreateHomebrew*Request` DTOs with Jakarta Validator
- [x] Integration tests with Testcontainers for `V15` migration and visibility query filtering in `player-character-service`
- [x] API integration tests via MockMvc for `POST /api/reference/**` happy path (201) and validation errors (`HomebrewCreationControllerTest`)
- [x] Security filter tests in `api-gateway` verifying 401 on missing JWT, 403 on non-DM role, and proper header propagation (`HomebrewGatewaySecurityTest`)
- [x] Frontend unit tests for `CreateHomebrewDialog` and DM button visibility using Vitest + React Testing Library (`__tests__/homebrew-creation.test.tsx`)
- [x] Build verification and test execution across all modules

---

## Documentation Tasks

- [x] Update OpenAPI spec annotations and endpoint bindings on new `POST /api/reference/**` endpoints
- [x] Verify alignment with `sdd-stack-contract.md` and Clean Architecture boundaries

---

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
