# Tasks: Add Homebrew Creation

> **Change ID:** `add-homebrew-creation`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### 1. Database & Persistence Layer
- [ ] Create Flyway migration `V8__enable_homebrew_ruleset.sql` in `player-character-service` adding `HOMEBREW` to CHECK constraints on `spell`, `feat`, `item`, `lineage`, and `archetype`
- [ ] Add `owner_id UUID NULL` column and composite indexes `(ruleset, owner_id)` to reference tables in `V8__enable_homebrew_ruleset.sql`
- [ ] Update JPA reference entities with `@Column(name = "owner_id") private UUID ownerId;`
- [ ] Update `ReferenceDataPersistenceAdapter` query implementations to support visibility filter `(owner_id IS NULL OR owner_id = :userId)`

### 2. Domain & Application Layer
- [ ] Update `Ruleset.java` in `player-character-service` domain layer to include `HOMEBREW`
- [ ] Create inbound use case port `CreateHomebrewUseCase` in `application.port.in`
- [ ] Implement request DTOs with Jakarta Validation in `application.dto`:
  - [ ] `CreateHomebrewSpellRequest`
  - [ ] `CreateHomebrewItemRequest`
  - [ ] `CreateHomebrewFeatRequest`
  - [ ] `CreateHomebrewLineageRequest`
  - [ ] `CreateHomebrewArchetypeRequest`
- [ ] Implement `HomebrewService` in `application.service` implementing `CreateHomebrewUseCase`

### 3. Web Layer & API Gateway
- [ ] Implement `POST /api/reference/{category}` endpoints in `ReferenceDataController` returning HTTP 201 with `Location` header
- [ ] Extract `X-User-Id` header in `ReferenceDataController` and pass creator ID to use cases and visibility queries
- [ ] Update `JwtHeaderFilter` in `api-gateway` to require `ROLE_DM` for `POST /api/reference/**` routes
- [ ] Configure `api-gateway` to strip incoming `X-User-Id` / `X-User-Roles` from external clients and propagate verified values from JWT claims

### 4. Frontend Integration
- [ ] Update frontend types in `lib/api/characters.ts` to include `HOMEBREW` in `Ruleset` type
- [ ] Create TanStack Query mutation hooks and API client methods in `lib/api/homebrew.ts`
- [ ] Create `CreateHomebrewDialog` component with Zod validation and Portuguese (`pt-BR`) labels in `components/homebrew/create-homebrew-dialog.tsx`
- [ ] Integrate "Criar novo" action button in `app/compendium/spells/page.tsx` with DM role guard (`roles.includes('DM')`)
- [ ] Add success toast notification and cache invalidation on mutation success

---

## Test Tasks

- [ ] Unit tests for `HomebrewService` and domain logic using JUnit 5 + Mockito
- [ ] Validation tests for `CreateHomebrew*Request` DTOs with Jakarta Validator
- [ ] Integration tests with Testcontainers for `V8` migration and visibility query filtering in `player-character-service`
- [ ] API integration tests via MockMvc for `POST /api/reference/**` happy path (201) and validation errors (400)
- [ ] Security filter tests in `api-gateway` verifying 401 on missing JWT, 403 on non-DM role, and proper header propagation
- [ ] Frontend unit tests for `CreateHomebrewDialog` and DM button visibility using Vitest + React Testing Library
- [ ] JaCoCo verification verifying ≥ 80% line coverage on new domain and application classes

---

## Documentation Tasks

- [ ] Update OpenAPI spec annotations on new `POST /api/reference/**` endpoints
- [ ] Verify alignment with `sdd-stack-contract.md` and Clean Architecture boundaries

---

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
