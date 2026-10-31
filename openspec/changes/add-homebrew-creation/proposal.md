# Proposal: Add Homebrew Creation

> **Change ID:** `add-homebrew-creation`
> **Status:** draft
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Implement the capability for Dungeon Masters (DM role) to create custom Homebrew reference content (Archetypes, Spells, Feats, Items, Lineages) tagged with ruleset `HOMEBREW`. Includes a "Criar novo" dialog button in each encyclopedia section on the frontend, gated to DM users, with backend persistence, strict owner isolation (`owner_id = user_id`), and role-based gateway authorization. Future campaign sharing remains out of scope.

## Motivation

- **User & DM Flexibility:** DMs running customized campaigns need the ability to define homebrew spells, archetypes, feats, equipment, and lineages directly within the platform.
- **Ruleset Architecture Realization:** The Dungeoneer architecture specifies a Dual Ruleset Engine supporting `SRD_2014`, `SRD_2024`, and `HOMEBREW`. While the SRD options are seeded globally, `HOMEBREW` entities must be dynamically created and owned by individual creators.
- **Content Isolation:** Unshared homebrew content must not leak into standard reference lists for other users. Each homebrew item must be strictly scoped to its creator (`owner_id`) until campaign-level sharing is introduced in Phase 2/3.

## Scope

### In Scope

- **Database Migration:** Flyway migration `V8__enable_homebrew_ruleset.sql` in `player-character-service` adding `HOMEBREW` to reference data `ruleset` CHECK constraints, adding nullable `owner_id` (UUID) columns, and creating composite indexes `(ruleset, owner_id)`.
- **Domain & Application Services:** Expand `Ruleset` enum with `HOMEBREW`; define inbound use case port `CreateHomebrewUseCase` and outbound repository ports supporting creator isolation; implement application service with defensive input handling.
- **REST Endpoints:** `POST /api/reference/{category}` endpoints (`archetypes`, `spells`, `feats`, `items`, `lineages`) accepting validated DTOs, recording creator identity from `X-User-Id`, and returning HTTP 201 Created with `Location` header.
- **Query Isolation:** Adjust reference listing endpoints (`GET /api/reference/**`) to accept optional `X-User-Id` and filter queries using `(owner_id IS NULL OR owner_id = :userId)`.
- **API Gateway Authorization:** Update `JwtHeaderFilter` in `api-gateway` to require `ROLE_DM` for `POST /api/reference/**` mutations, and forward sanitized `X-User-Id` and `X-User-Roles` downstream.
- **Frontend UI & State:** "Criar novo" button in encyclopedia pages visible only to DMs; modal dialogs with Portuguese (pt-BR) labels; React Hook Form + Zod schema validation; TanStack Query mutation with cache invalidation and toast feedback.

### Out of Scope

- Monsters and NPC homebrew creation (belongs to `campaign-service`).
- Sharing homebrew items with specific players or campaigns (scheduled for campaign association phase).
- Homebrew item modification (`PUT`/`PATCH`) and deletion (`DELETE`) — subject to a follow-up RFC.
- Public marketplace or community publishing of homebrew content.

## Proposed Solution

### Approach

The solution follows Clean Architecture principles in `player-character-service`, gateway-level security enforcement in `api-gateway`, and idiomatic Next.js 16 components with TanStack Query on the frontend.

1. **Persistence & Schema Evolution:**
   A new Flyway migration `V8__enable_homebrew_ruleset.sql` will alter existing reference tables (`archetype`, `spell`, `feat`, `item`, `lineage`) to allow `HOMEBREW` in the `ruleset` CHECK constraint. It adds a nullable `owner_id UUID` column (null for system SRD content, populated for user-created homebrew) and adds composite indexes `(ruleset, owner_id)` to ensure performant listing queries.

2. **Domain & Hexagonal Boundaries:**
   The `domain/model/Ruleset.java` enum is extended to include `HOMEBREW`. A clean inbound port `CreateHomebrewUseCase` is introduced along with strictly validated application DTOs (e.g., `CreateHomebrewSpellRequest`). Domain entities remain completely free of Spring/JPA annotations. The persistence adapter implements custom queries filtering by `owner_id IS NULL OR owner_id = :userId`.

3. **API Gateway Security:**
   The `JwtHeaderFilter` in `api-gateway` intercepts mutation requests targeting `POST /api/reference/**`. It validates that a valid JWT token is present and that the claims include `ROLE_DM`. Upon authorization, the gateway strips any caller-supplied `X-User-Id` or `X-User-Roles` headers and injects verified values from JWT claims before proxying downstream. Read queries (`GET /api/reference/**`) permit anonymous access while forwarding `X-User-Id` if a valid JWT is provided.

4. **Frontend Encyclopedia Integration:**
   Frontend pages for reference data query the current user's session roles via auth state. If the user possesses the `DM` role, a "Criar novo" button is rendered above the encyclopedia data grid. Clicking the button opens a modal dialog tailored to the reference type, with client-side Zod validation and Portuguese (`pt-BR`) field labels. Successful mutations trigger toast notifications and invalidate the corresponding TanStack Query cache key to immediately reflect newly created homebrew items.

### Impacted Files

| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V8__enable_homebrew_ruleset.sql` | Create | Migration adding `HOMEBREW` to `ruleset` CHECK constraints and `owner_id` column to reference tables |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Ruleset.java` | Modify | Add `HOMEBREW` constant to ruleset enum |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/CreateHomebrewUseCase.java` | Create | Inbound use case port interface for homebrew creation |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewSpellRequest.java` | Create | DTO with Jakarta Validation for homebrew spells |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewItemRequest.java` | Create | DTO with Jakarta Validation for homebrew items |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewFeatRequest.java` | Create | DTO with Jakarta Validation for homebrew feats |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewLineageRequest.java` | Create | DTO with Jakarta Validation for homebrew lineages |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewArchetypeRequest.java` | Create | DTO with Jakarta Validation for homebrew archetypes |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java` | Create | Application service implementing `CreateHomebrewUseCase` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java` | Modify | Add `POST /api/reference/{category}` endpoints and handle `X-User-Id` for visibility |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java` | Modify | Support saving homebrew entities with `owner_id` and querying with visibility predicate |
| `api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java` | Modify | Protect `POST /api/reference/**`, enforce `ROLE_DM`, sanitize and inject `X-User-Id` and `X-User-Roles` |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | Update client TypeScript types to accept `HOMEBREW` ruleset |
| `dungeoneer-frontend/lib/api/homebrew.ts` | Create | TanStack Query mutation hooks and API client functions for homebrew |
| `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` | Create | Reusable DM-gated modal dialog with category-specific forms |
| `dungeoneer-frontend/app/compendium/spells/page.tsx` | Modify | Integrate "Criar novo" action button and homebrew listing integration |

### Data Model Changes (if applicable)

Tables affected in `player_character` schema:
- `spell`, `feat`, `item`, `lineage`, `archetype`

DDL Migration overview (`V8__enable_homebrew_ruleset.sql`):
```sql
-- Update ruleset CHECK constraint to include 'HOMEBREW'
ALTER TABLE player_character.spell DROP CONSTRAINT IF EXISTS chk_spell_ruleset;
ALTER TABLE player_character.spell ADD CONSTRAINT chk_spell_ruleset CHECK (ruleset IN ('SRD_2014', 'SRD_2024', 'HOMEBREW'));

ALTER TABLE player_character.feat DROP CONSTRAINT IF EXISTS chk_feat_ruleset;
ALTER TABLE player_character.feat ADD CONSTRAINT chk_feat_ruleset CHECK (ruleset IN ('SRD_2014', 'SRD_2024', 'HOMEBREW'));

ALTER TABLE player_character.item DROP CONSTRAINT IF EXISTS chk_item_ruleset;
ALTER TABLE player_character.item ADD CONSTRAINT chk_item_ruleset CHECK (ruleset IN ('SRD_2014', 'SRD_2024', 'HOMEBREW'));

ALTER TABLE player_character.lineage DROP CONSTRAINT IF EXISTS chk_lineage_ruleset;
ALTER TABLE player_character.lineage ADD CONSTRAINT chk_lineage_ruleset CHECK (ruleset IN ('SRD_2014', 'SRD_2024', 'HOMEBREW'));

ALTER TABLE player_character.archetype DROP CONSTRAINT IF EXISTS chk_archetype_ruleset;
ALTER TABLE player_character.archetype ADD CONSTRAINT chk_archetype_ruleset CHECK (ruleset IN ('SRD_2014', 'SRD_2024', 'HOMEBREW'));

-- Add owner_id to reference tables
ALTER TABLE player_character.spell ADD COLUMN IF NOT EXISTS owner_id UUID NULL;
ALTER TABLE player_character.feat ADD COLUMN IF NOT EXISTS owner_id UUID NULL;
ALTER TABLE player_character.item ADD COLUMN IF NOT EXISTS owner_id UUID NULL;
ALTER TABLE player_character.lineage ADD COLUMN IF NOT EXISTS owner_id UUID NULL;
ALTER TABLE player_character.archetype ADD COLUMN IF NOT EXISTS owner_id UUID NULL;

-- Indexes for efficient visibility queries: (ruleset, owner_id)
CREATE INDEX IF NOT EXISTS idx_spell_owner_ruleset ON player_character.spell (ruleset, owner_id);
CREATE INDEX IF NOT EXISTS idx_feat_owner_ruleset ON player_character.feat (ruleset, owner_id);
CREATE INDEX IF NOT EXISTS idx_item_owner_ruleset ON player_character.item (ruleset, owner_id);
CREATE INDEX IF NOT EXISTS idx_lineage_owner_ruleset ON player_character.lineage (ruleset, owner_id);
CREATE INDEX IF NOT EXISTS idx_archetype_owner_ruleset ON player_character.archetype (ruleset, owner_id);
```

## Acceptance Criteria

- [ ] **Given** a user authenticated with role `ROLE_DM`, **When** they submit a valid payload to `POST /api/reference/spells` with custom spell details, **Then** the service responds with HTTP 201 Created, sets `ruleset = 'HOMEBREW'`, records the user's UUID in `owner_id`, and returns the created spell with a generated UUID and `Location` header.
- [ ] **Given** a user authenticated with role `ROLE_PLAYER` (lacking `ROLE_DM`), **When** they attempt `POST /api/reference/spells`, **Then** the API Gateway intercepts the request and responds with HTTP 403 Forbidden before reaching the downstream microservice.
- [ ] **Given** an unauthenticated request without a valid JWT token, **When** they attempt `POST /api/reference/spells`, **Then** the API Gateway responds with HTTP 401 Unauthorized.
- [ ] **Given** two distinct DM users (User A and User B), **When** User A queries `GET /api/reference/spells` sending their authenticated header, **Then** the response includes standard SRD spells and User A's created homebrew spells, but excludes User B's homebrew spells.
- [ ] **Given** an unauthenticated public query to `GET /api/reference/spells` (no `X-User-Id`), **When** the request is processed, **Then** only global SRD spells where `owner_id IS NULL` are returned.
- [ ] **Given** a DM user submitting a homebrew item with an invalid payload (e.g. blank name or negative weight), **When** `POST /api/reference/items` is invoked, **Then** the service returns HTTP 400 Bad Request with an RFC 7807 `ProblemDetail` response detailing validation violations.
- [ ] **Given** an authenticated user who is not a DM viewing the encyclopedia page, **When** the page renders, **Then** the "Criar novo" action button is hidden.
- [ ] **Given** a DM user on the spells encyclopedia page, **When** they click "Criar novo", fill out the dialog with valid inputs, and confirm submission, **Then** a TanStack Query mutation executes, a success toast in Portuguese is displayed, the modal closes, and the table automatically invalidates and displays the new item.

## Defensive Security & Validation Considerations

### Input Validation

- All incoming DTOs (`CreateHomebrewSpellRequest`, `CreateHomebrewItemRequest`, etc.) are validated at the controller boundary using Jakarta Validation annotations (`@NotBlank`, `@Size(max = 255)`, `@Min(0)`, `@Max(9)`, `@NotNull`).
- Text and description fields allow markdown but are sanitized against Cross-Site Scripting (XSS) attacks by escaping raw HTML tags before storage and rendering.
- Frontend forms use Zod schemas matching backend constraints to provide instant inline validation feedback.

### Secret Isolation

- No tokens, passwords, or database secrets are hardcoded in code or configuration files.
- JWT verification keys and database credentials are read from environment variables (`JWT_SECRET_KEY`, `DB_PASSWORD`).

### Authorization & Authentication Impact

- Role-based Access Control (RBAC) is enforced at the `api-gateway` level via `JwtHeaderFilter`.
- Only requests bearing a verified JWT claim with `ROLE_DM` can access mutation routes `POST /api/reference/**`.
- Inbound HTTP headers `X-User-Id` and `X-User-Roles` sent by external clients are stripped at the gateway and replaced with validated claims from the JWT to prevent header spoofing.
- Downstream microservices trust gateway headers for tenant/owner scoping.

### Other Security Risks

- **SQL Injection:** Mitigated by relying strictly on Spring Data JPA parameterized queries and Criteria API predicates for dynamic visibility filtering.
- **Resource Exhaustion (Denial of Service):** Payloads have strict `@Size` limits (e.g., description max 4000 characters). Future rate limiting can be applied at the gateway level.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests

| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | DM creates homebrew spell with valid fields | Integration (Testcontainers) | HTTP 201 Created, `owner_id` matching `X-User-Id`, `ruleset=HOMEBREW` |
| TP-02 | DM lists spells with `X-User-Id` present | Integration (Testcontainers) | Returns SRD spells + owned homebrew spells |
| TP-03 | Unauthenticated client lists reference items | Integration (MockMvc) | Returns only SRD items (`owner_id IS NULL`) |
| TP-04 | Use case executes homebrew entity creation | Unit (JUnit 5 + Mockito) | Domain model validated and repository port invoked with mapped entity |
| TP-05 | DM clicks "Criar novo" and submits form | Frontend Unit (Vitest + RTL) | Dialog opens, mutation fired, toast displayed, modal dismissed |

### Edge Case & Failure Mode Tests

| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Player without DM role attempts POST homebrew | Integration (MockMvc / Gateway) | HTTP 403 Forbidden |
| TE-02 | Missing or malformed JWT on POST homebrew | Integration (MockMvc / Gateway) | HTTP 401 Unauthorized |
| TE-03 | Invalid DTO (blank name, level > 9) | Integration (MockMvc) | HTTP 400 Bad Request with RFC 7807 `ProblemDetail` |
| TE-04 | User queries reference data belonging to another DM | Integration (Testcontainers) | Other DM's homebrew entities are omitted from result list |
| TE-05 | Client attempts to spoof `X-User-Id` header | Integration (Gateway test) | Gateway overwrites spoofed header with authenticated JWT subject |
| TE-06 | Database constraint violation during creation | Unit / Integration | Domain exception mapped to RFC 7807 409 Conflict or 400 Bad Request |

### Test Commands

- **Backend Tests:** `mvn clean test` (within microservice directory or `mvn -f player-character-service/backend/pom.xml clean test`)
- **Backend Coverage Check:** `mvn -f player-character-service/backend/pom.xml clean verify` (enforcing ≥ 80% line coverage on domain and application layers)
- **Gateway Tests:** `mvn -f api-gateway/backend/pom.xml clean test`
- **Frontend Unit Tests:** `cd dungeoneer-frontend && npx vitest run`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Reference data queries degrade in performance with mixed SRD and Homebrew data | Low | Med | Added composite indexes `(ruleset, owner_id)` and indexed foreign keys in PostgreSQL |
| Header spoofing of `X-User-Id` bypassing ownership checks | Med | High | API Gateway strips incoming caller-supplied `X-User-Id` and `X-User-Roles` headers before injecting claims |
| High memory consumption if users create excessively large text descriptions | Low | Low | Strict `@Size(max = 4000)` Jakarta validation annotations on all text fields |
| User confusion if homebrew content is not visually distinguished | Med | Low | Frontend badge with `Homebrew` label and distinct visual styling in tables |

## Specification Changes & Delta Stubs

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Flyway migration `V8__enable_homebrew_ruleset.sql` schema changes; `ruleset` CHECK constraint expansion (`HOMEBREW`); `owner_id` column addition; composite indexes `(ruleset, owner_id)`.

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Expansion of `Ruleset` enum with `HOMEBREW`; definition of `CreateHomebrewUseCase` inbound port; DTO schemas; repository query visibility contract (`owner_id IS NULL OR owner_id = :userId`).

### `specs/system-architecture-delta.md`
- **Target Canonical Spec:** `openspec/specs/system-architecture.md`
- **Summary of Updates:** API Gateway route security rules for `POST /api/reference/**` enforcing `ROLE_DM`; stripping and injection of `X-User-Id` and `X-User-Roles` headers.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Encyclopedia "Criar novo" DM action button; modal creation dialogs; Zod schemas; TanStack Query mutations and cache invalidation.

## References

- [sdd-stack-contract.md](../../specs/sdd-stack-contract.md)
- [system-architecture.md](../../specs/system-architecture.md)
- [player-character-domain.md](../../specs/player-character-domain.md)
- [player-character-data-model.md](../../specs/player-character-data-model.md)
- [frontend-web-app.md](../../specs/frontend-web-app.md)
