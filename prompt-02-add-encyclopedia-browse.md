Focus: Full-Stack / Spring Boot 4 (Java 21 / Clean Architecture / PostgreSQL) & Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Execute Full Phased Implementation of Change `add-encyclopedia-browse`

## Execution Context
- Active Plan: `openspec/changes/add-encyclopedia-browse/tasks.md`
- Linked Proposal: `openspec/changes/add-encyclopedia-browse/proposal.md`
- Delta Specs:
  - `openspec/changes/add-encyclopedia-browse/specs/player-character-data-model-delta.md`
  - `openspec/changes/add-encyclopedia-browse/specs/player-character-domain-delta.md`
  - `openspec/changes/add-encyclopedia-browse/specs/frontend-web-app-delta.md`
- Dependency Prerequisite: Change `add-main-dashboard` (Prompt 01) must be implemented and committed before starting this task, as the encyclopedia browse views connect to the navigation categories registered in the main dashboard.
- Authentication Context: User authentication is already active in the application with roles `PLAYER` and `DUNGEON_MASTER`. Reference data browse is accessible to all authenticated users. (Note: The mention of "no auth" in `AGENTS.md` is outdated).
- Core Objectives:
  1. Complete the placeholder sections of `proposal.md` (Given-When-Then acceptance criteria, defensive security against FTS injection and pagination abuse, and shift-left testing strategy) and populate `tasks.md`. Await user confirmation before writing code.
  2. Implement Flyway migration `V7__create_spells_feats_items_and_fts_indexes.sql` creating tables for `spells`, `feats`, and `items`, along with `tsvector` columns and GIN indexes for PostgreSQL full-text search across name and description (and ensure FTS indexes on `archetypes` and `lineages`).
  3. Implement backend domain model (`Item.java`), outbound repository ports, JPA entities, and Spring Data repositories supporting cursor-based pagination and native FTS queries.
  4. Implement application use cases, services, DTOs (`CursorPageResponse<T>`), and REST endpoints on `ReferenceDataController` with cursor, search query (`q`), ruleset filter, and domain-specific filters (spell level, item rarity, etc.).
  5. Build frontend API integration in `lib/api/encyclopedia.ts` utilizing TanStack Query `useInfiniteQuery` for lazy cursor fetching.
  6. Create responsive Next.js 16 encyclopedia browse pages for Classes/Archetypes, Spells, Feats, Items, and Lineages with search, filters, infinite scroll, loading skeletons, and empty states.
  7. SEED SCOPE (added): Inject the full SRD content for Spells, Feats and Items (equipment, weapons, armor, magic items), translated to pt-BR, into the database via versioned Flyway migrations (V8 spells, V9 feats, V10 items), for BOTH SRD 5.1 (`SRD_2014`) and SRD 5.2 (`SRD_2024`). Content is static, read-only text for encyclopedia consultation. Archetypes and Lineages are already seeded; do not re-seed them.
- Seed Rules (added):
  - Source: SRD 5.1 and SRD 5.2 (CC-BY 4.0). Do NOT invent content; use only official SRD text. Add the required CC-BY attribution to `NOTICE`/README and to the encyclopedia UI footer.
  - Translation: produced by you (LLM) into pt-BR, stored in versioned data files (JSON) under `player-character-service/backend/src/main/resources/db/seed/` and mirrored in the SQL migrations. Keep official D&D pt-BR terminology where established. Keep the original English name in `name_en`; `slug` derives from the English name.
  - Idempotency: every INSERT uses `ON CONFLICT (ruleset, slug) DO NOTHING`; `UNIQUE (ruleset, slug)` on each table.
  - Batching: one migration per content type, large files split by chunk (e.g. spells by level) so the user can review by sampling. Never put secrets or environment-specific data in seeds.
  - Review gate: after each seed phase, produce a pt-BR sample report (10 random records with EN original vs PT translation) and wait for user review before committing.
  - Safety: escape SQL properly (dollar-quoting `$$...$$` or parameterized loading); no string-built SQL from untrusted input.
- Architectural & Quality Constraints:
  - SDD Spec-First Gate: Never write implementation code while `proposal.md`, delta specs, and `tasks.md` contain placeholders. Obtain explicit user confirmation at the end of Phase 1.
  - Clean Architecture (Hexagonal): Backend `domain/model/` must contain ZERO framework imports (no Spring, no JPA, no Lombok). No DTO annotations in domain entities.
  - Defensive Security: Sanitize full-text search parameters against query injection using parameterized queries with PostgreSQL `plainto_tsquery` or `websearch_to_tsquery`. Enforce maximum pagination limit (`@Max(100)`) on cursor requests.
  - TDD & Quality Assurance: Write unit and integration tests (using global Maven and Testcontainers PostgreSQL for backend, Vitest for frontend). Enforce >= 80% JaCoCo coverage on domain and application layers.
  - Maven Tool Rule: ALWAYS use global Maven (`mvn`), NEVER the wrapper script (do not run `./mvnw` or `.\mvnw`).
  - Language Separation: Backend Java code, DTOs, domain models, commit messages, and automated tests strictly in English. Frontend user-facing UI labels, filter selectors, empty state messages, and shared walkthrough strictly in Portuguese (`pt-BR`).
  - Phase Pauses & Ready Commits: Execute sequentially through phases. Pause after each phase, update the cumulative walkthrough (`openspec/changes/add-encyclopedia-browse/walkthrough.md` in pt-BR), provide the exact ready-to-run Conventional Commit command, and await user confirmation.

---

### Phase 1: Spec Completion, Dependency Check & User Approval Gate

#### Execution Context
- Scope: Execute Phase 1 only. Verify that prerequisite change `add-main-dashboard` has been completed. Finalize the SDD proposal, delta specifications, and task breakdown. Do NOT modify application source code during this phase.

#### Implementation Steps
1. Prerequisite Verification:
   - Check that `prompt-01-add-main-dashboard` has been executed (verify presence of `EncyclopediaMenu` and routes).
2. Inspect `openspec/changes/add-encyclopedia-browse/proposal.md` and complete all placeholder sections:
   - Acceptance Criteria: Formulate Given-When-Then scenarios covering:
      * Seed: after migrations, spells/feats/items exist for both rulesets with pt-BR `name`/`description` and `name_en`; re-running migrations is idempotent (no duplicates); minimum record counts per ruleset/type are asserted by tests.
      * Search on seeded data works by pt-BR text and by original English name (`name_en`).
     * Cursor-based pagination: initial page fetch, subsequent page fetch with `cursor`, end of list indicator (`hasNext: false`).
     * Full-text search: searching "míssil" matches "Mísseis Mágicos" across name and description.
     * Filtering: filtering spells by level or school; filtering items by rarity; filtering by ruleset (`SRD_2014`, `SRD_2024`).
     * Frontend infinite scroll loading seamlessly without UI stutter.
   - Defensive Security & Input Validation:
     * Validation of query parameters (`limit` between 1 and 100, safe sanitization of search text `q`).
     * Secret isolation: Confirmation of environment variable usage for database credentials.
     * Authorization impact: All authenticated users (`PLAYER` and `DUNGEON_MASTER`) have read access to reference data.
     * SQL injection mitigation on tsquery expressions.
   - Testing Strategy & Shift-Left Plan: Detail happy path and edge-case test matrices across backend (MockMvc, Testcontainers) and frontend (Vitest).
3. Complete delta specifications under `openspec/changes/add-encyclopedia-browse/specs/`:
   - `player-character-data-model-delta.md`: Detail new tables (`spells`, `feats`, `items`), columns, and GIN tsvector indexes.
   - `player-character-domain-delta.md`: Detail domain entity `Item` and query port extensions.
   - `frontend-web-app-delta.md`: Detail browse routes, infinite query hooks, and filter components.
4. Populate `openspec/changes/add-encyclopedia-browse/tasks.md`:
   - Detail atomic checkable tasks for implementation, testing, and documentation.
5. Present the completed proposal and tasks to the user and request explicit confirmation before writing code.

#### Quality Gates & Verification
- Verify completeness of markdown files in `openspec/changes/add-encyclopedia-browse/`.
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/add-encyclopedia-browse/proposal.md \
          openspec/changes/add-encyclopedia-browse/tasks.md \
          openspec/changes/add-encyclopedia-browse/specs/
  git commit -m "docs(spec): complete proposal and tasks for add-encyclopedia-browse"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Initialize `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. Summary of specification details, data model additions, and acceptance criteria.
  2. Await user approval on the spec before advancing to Phase 2.

---

### Phase 2: Database Migration (Flyway V7) & PostgreSQL Full-Text Search Indexes

#### Execution Context
- Scope: Execute Phase 2 only. Create the Flyway migration script `V7__create_spells_feats_items_and_fts_indexes.sql` defining relational tables for spells, feats, and items, with PostgreSQL full-text search indexes.

#### Implementation Steps
1. Create Flyway migration script in `player-character-service/backend/src/main/resources/db/migration/`:
   - Name: `V7__create_spells_feats_items_and_fts_indexes.sql`.
   - Table `player_character.spells`:
      * Seed-extended columns: `slug VARCHAR(150) NOT NULL`, `name_en VARCHAR(150) NOT NULL`, `UNIQUE (ruleset, slug)`, in addition to the columns below. Same `slug`, `name_en` and UNIQUE on `feats` and `items`; `items` also gets `item_category VARCHAR(30) NOT NULL` (EQUIPMENT/WEAPON/ARMOR/MAGIC). Include `name_en` in the tsvector expressions.
     * Columns: `id UUID PRIMARY KEY`, `ruleset VARCHAR(20) NOT NULL`, `name VARCHAR(150) NOT NULL`, `level INT NOT NULL`, `school VARCHAR(50) NOT NULL`, `casting_time VARCHAR(50)`, `range_area VARCHAR(100)`, `components VARCHAR(50)`, `duration VARCHAR(100)`, `description TEXT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
   - Table `player_character.feats`:
     * Columns: `id UUID PRIMARY KEY`, `ruleset VARCHAR(20) NOT NULL`, `name VARCHAR(150) NOT NULL`, `prerequisites VARCHAR(255)`, `description TEXT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
   - Table `player_character.items`:
     * Columns: `id UUID PRIMARY KEY`, `ruleset VARCHAR(20) NOT NULL`, `name VARCHAR(150) NOT NULL`, `item_type VARCHAR(50) NOT NULL`, `rarity VARCHAR(30) NOT NULL`, `attunement BOOLEAN NOT NULL DEFAULT FALSE`, `cost VARCHAR(50)`, `weight VARCHAR(30)`, `description TEXT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
   - Full-Text Search Indexes (using GIN with `to_tsvector`):
     * GIN index on `to_tsvector('portuguese', name || ' ' || description)` for `player_character.spells`.
     * GIN index on `to_tsvector('portuguese', name || ' ' || description)` for `player_character.feats`.
     * GIN index on `to_tsvector('portuguese', name || ' ' || description)` for `player_character.items`.
     * GIN indexes for existing `player_character.archetypes` and `player_character.lineages` on name and description.
   - Standard B-Tree indexes for filtering and cursor ordering:
     * `(ruleset, id)` and `(level, id)` on spells.
     * `(ruleset, id)` and `(item_type, id)` on items.
     * `(ruleset, id)` on feats, archetypes, and lineages.

#### Quality Gates & Verification
- Execute database migration test using global Maven:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=*MigrationTest*
  ```
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/V7__create_spells_feats_items_and_fts_indexes.sql
  git commit -m "feat(db): add V7 Flyway migration for spells, feats, items and FTS GIN indexes"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2 section to `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. Tables created, schema definitions, and GIN index strategies.
  2. Migration test outputs.
  3. Pause and present the commit command to the user before proceeding to Phase 3.

---

### Phase 2.1: SRD Seed — Spells (Flyway V8)

#### Execution Context
- Scope: Execute Phase 2.1 only. Seed ALL SRD spells for `SRD_2014` and `SRD_2024`, translated to pt-BR. Follow the "Seed Rules" in Execution Context.

#### Implementation Steps
1. Build source data files in `player-character-service/backend/src/main/resources/db/seed/spells_srd_2014.json` and `spells_srd_2024.json` (fields: slug, name_en, name, level, school, casting_time, range_area, components, duration, description).
2. Create `V8__seed_srd_spells.sql` (split into several V8 chunks by spell level if too large, e.g. `V8_1__...`) using `INSERT ... ON CONFLICT (ruleset, slug) DO NOTHING` and dollar-quoted descriptions.
3. Write Testcontainers test `SpellSeedMigrationTest`: minimum counts per ruleset, no duplicate (ruleset, slug), no blank pt-BR fields, FTS finds a spell by pt-BR term and by `name_en`, re-applying the seed is idempotent.
4. Produce a pt-BR sample report (10 random spells, EN vs PT) and wait for user review.

#### Quality Gates & Verification
- `mvn -f player-character-service/backend/pom.xml test -Dtest=*SeedMigrationTest*`
- Commit: `git add player-character-service/backend/src/main/resources/db/ player-character-service/backend/src/test/java/` then `git commit -m "feat(db): seed SRD 2014/2024 spells translated to pt-BR"`

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2.1 (pt-BR) to `walkthrough.md`: counts, translation criteria, sample report, test output. Pause for user review/commit.

---

### Phase 2.2: SRD Seed — Feats (Flyway V9)

#### Execution Context
- Scope: Execute Phase 2.2 only. Seed ALL SRD feats (SRD 5.2 has a full feat list; SRD 5.1 has very few — seed exactly what each SRD contains, do not invent) for both rulesets, translated to pt-BR.

#### Implementation Steps
1. Source files `feats_srd_2014.json` / `feats_srd_2024.json` (slug, name_en, name, prerequisites, description).
2. `V9__seed_srd_feats.sql`, idempotent as above.
3. Testcontainers test `FeatSeedMigrationTest` (counts, uniqueness, non-blank, FTS pt-BR + `name_en`, idempotency).
4. pt-BR sample report; wait for user review.

#### Quality Gates & Verification
- `mvn -f player-character-service/backend/pom.xml test -Dtest=*SeedMigrationTest*`
- Commit: `git add player-character-service/backend/src/main/resources/db/ player-character-service/backend/src/test/java/` then `git commit -m "feat(db): seed SRD 2014/2024 feats translated to pt-BR"`

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2.2 (pt-BR) to `walkthrough.md`. Pause for user review/commit.

---

### Phase 2.3: SRD Seed — Items (Flyway V10)

#### Execution Context
- Scope: Execute Phase 2.3 only. Seed ALL SRD items: adventuring gear/equipment, weapons, armor, and magic items, for both rulesets, translated to pt-BR. Load in chunks by `item_category` (e.g. `V10_1` equipment, `V10_2` weapons/armor, `V10_3` magic items).

#### Implementation Steps
1. Source files `items_srd_2014.json` / `items_srd_2024.json` (slug, name_en, name, item_category, item_type, rarity, attunement, cost, weight, description). Non-magic items use rarity `COMUM`/`NONE` consistently as defined in the delta spec.
2. Flyway migrations V10.x, idempotent as above.
3. Testcontainers test `ItemSeedMigrationTest` (counts per category/ruleset, uniqueness, non-blank, FTS pt-BR + `name_en`, filter by rarity/type returns expected rows, idempotency).
4. pt-BR sample report; wait for user review.

#### Quality Gates & Verification
- `mvn -f player-character-service/backend/pom.xml test -Dtest=*SeedMigrationTest*`
- Commit: `git add player-character-service/backend/src/main/resources/db/ player-character-service/backend/src/test/java/` then `git commit -m "feat(db): seed SRD 2014/2024 items translated to pt-BR"`

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2.3 (pt-BR) to `walkthrough.md`, plus the CC-BY attribution added to NOTICE/README. Pause for user review/commit before Phase 3.

---

### Phase 3: Backend Domain Models & Persistence Adapters (Clean Architecture)

#### Execution Context
- Scope: Execute Phase 3 only. Implement pure domain models, outbound repository ports, JPA entities, and Spring Data repositories supporting cursor queries and PostgreSQL full-text search.

#### Implementation Steps
1. Domain Layer (`domain/model/` - zero framework imports, no Spring/JPA/Lombok):
   - Create pure domain entity `Item.java`:
     * Attributes: `id` (UUID), `ruleset` (Ruleset enum), `name` (String), `itemType` (String), `rarity` (String), `attunement` (boolean), `cost` (String), `weight` (String), `description` (String).
     * Invariants: `name` non-blank, `ruleset` non-null, `itemType` non-blank. Violations throw `CharacterValidationException`.
   - Verify existing domain entities `Spell.java`, `Feat.java`, `Archetype.java`, `Lineage.java` and ensure they align with reference queries.
2. Outbound Repository Ports (`application/port/out/`):
   - Update or extend `ReferenceDataRepository.java` to support paginated cursor queries with optional search query and filters:
     * `findSpells(UUID afterCursor, int limit, String query, Ruleset ruleset, Integer level, String school)`
     * `findFeats(UUID afterCursor, int limit, String query, Ruleset ruleset)`
     * `findItems(UUID afterCursor, int limit, String query, Ruleset ruleset, String itemType, String rarity)`
     * `findArchetypes(UUID afterCursor, int limit, String query, Ruleset ruleset)`
     * `findLineages(UUID afterCursor, int limit, String query, Ruleset ruleset)`
3. Persistence Layer (`adapter/out/persistence/`):
   - Create JPA entities: `ItemJpaEntity.java`, `SpellJpaEntity.java`, `FeatJpaEntity.java`.
   - Create Spring Data interfaces: `SpringDataSpellRepository`, `SpringDataFeatRepository`, `SpringDataItemRepository`.
   - Update `SpringDataArchetypeRepository` and `SpringDataLineageRepository`.
   - Implement native or JPQL queries using parameterized `websearch_to_tsquery('portuguese', :q)` or `plainto_tsquery` and cursor condition `(:afterCursor IS NULL OR e.id > :afterCursor) ORDER BY e.id ASC LIMIT :limit`.
   - Implement / update `ReferenceDataPersistenceAdapter.java` implementing `ReferenceDataRepository`.
4. Tests:
   - Unit tests for domain models (`ItemTest.java`).
   - Integration tests with Testcontainers PostgreSQL verifying cursor navigation and FTS query matches.

#### Quality Gates & Verification
- Execute backend tests using global Maven:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Item.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ \
          player-character-service/backend/src/test/java/
  git commit -m "feat(backend): add Item domain model and persistence adapters with cursor and FTS"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 3 section to `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. Domain entities and persistence methods implemented.
  2. Unit and Testcontainers test outputs.
  3. Pause and present the commit command to the user before proceeding to Phase 4.

---

### Phase 4: Backend Application Services, DTOs & REST Endpoints

#### Execution Context
- Scope: Execute Phase 4 only. Implement inbound use case ports, application service methods, DTOs with Jakarta validation, REST controller endpoints, and MockMvc tests.

#### Implementation Steps
1. Application DTOs (`application/dto/`):
   - Create generic `CursorPageResponse<T>` record:
     * `List<T> items`, `UUID nextCursor`, `boolean hasNext`, `int pageSize`.
   - Create DTO records: `SpellDto`, `FeatDto`, `ItemDto`, `ArchetypeSummaryDto`, `LineageSummaryDto`.
2. Inbound Use Cases (`application/port/in/`):
   - Update `GetReferenceDataUseCase` with paginated query methods for spells, feats, items, archetypes, and lineages.
3. Application Service (`application/service/ReferenceDataService.java`):
   - Implement cursor calculation logic: fetch `limit + 1` records; if result size > `limit`, set `hasNext = true` and `nextCursor = items.get(limit - 1).getId()`, returning sublist of size `limit`.
4. REST Web Controller (`adapter/in/web/controller/ReferenceDataController.java`):
   - Add/update endpoints:
     * `GET /api/reference/spells?cursor={uuid}&limit={limit}&q={q}&ruleset={ruleset}&level={lvl}&school={school}`
     * `GET /api/reference/feats?cursor={uuid}&limit={limit}&q={q}&ruleset={ruleset}`
     * `GET /api/reference/items?cursor={uuid}&limit={limit}&q={q}&ruleset={ruleset}&type={type}&rarity={rarity}`
     * `GET /api/reference/classes?cursor={uuid}&limit={limit}&q={q}&ruleset={ruleset}`
     * `GET /api/reference/lineages?cursor={uuid}&limit={limit}&q={q}&ruleset={ruleset}`
   - Validate pagination parameters: `@Min(1) @Max(100) int limit` (defaulting to 20).
   - Return RFC 7807 `ProblemDetail` on validation failure.
5. Integration & Unit Tests:
   - MockMvc tests in `ReferenceDataControllerTest`:
     * Verify 200 OK with `CursorPageResponse`.
     * Verify cursor continuation: pass returned `nextCursor` to subsequent request and assert consecutive items.
     * Verify search parameter filtering.
     * Verify 400 Bad Request on invalid `limit` (> 100).

#### Quality Gates & Verification
- Execute full backend test suite and verify JaCoCo coverage (>= 80% on domain and application layers):
  ```bash
  mvn -f player-character-service/backend/pom.xml clean verify
  ```
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/ \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/ \
          player-character-service/backend/src/test/java/
  git commit -m "feat(backend): add cursor paginated REST endpoints and DTOs for encyclopedia"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 4 section to `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. REST API contract definitions and DTO structures.
  2. MockMvc and JaCoCo coverage test report.
  3. Pause and present the commit command to the user before proceeding to Phase 5.

---

### Phase 5: Frontend API Integration & TanStack Infinite Query Hooks

#### Execution Context
- Scope: Execute Phase 5 only. Implement API client functions and TanStack Query `useInfiniteQuery` hooks for each encyclopedia category.

#### Implementation Steps
1. Create `dungeoneer-frontend/lib/api/encyclopedia.ts`:
   - TypeScript interfaces:
     * `CursorPageResponse<T>` (`items: T[]`, `nextCursor: string | null`, `hasNext: boolean`, `pageSize: number`).
     * `Spell`, `Feat`, `Item`, `Archetype`, `Lineage`.
     * Query filter parameter types (`SpellFilters`, `ItemFilters`, `CommonFilters`).
   - Query client fetchers:
     * `fetchSpells(params: SpellFilters & { cursor?: string; limit?: number })`
     * `fetchFeats(params: CommonFilters & { cursor?: string; limit?: number })`
     * `fetchItems(params: ItemFilters & { cursor?: string; limit?: number })`
     * `fetchArchetypes(params: CommonFilters & { cursor?: string; limit?: number })`
     * `fetchLineages(params: CommonFilters & { cursor?: string; limit?: number })`
   - TanStack React Query custom hooks:
     * `useInfiniteSpells(filters)`
     * `useInfiniteFeats(filters)`
     * `useInfiniteItems(filters)`
     * `useInfiniteArchetypes(filters)`
     * `useInfiniteLineages(filters)`
     * Configure `getNextPageParam: (lastPage) => lastPage.hasNext ? lastPage.nextCursor : undefined`.
2. Write unit tests in `dungeoneer-frontend/__tests__/encyclopedia-api.test.ts`:
   - Test URL query string formation with search, cursor, and filters.
   - Test `getNextPageParam` behavior with and without subsequent cursor.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/lib/api/encyclopedia.ts \
          dungeoneer-frontend/__tests__/encyclopedia-api.test.ts
  git commit -m "feat(frontend): create TanStack Query infinite hooks for encyclopedia reference data"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 5 section to `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. API client functions and hooks created.
  2. Unit test verification results.
  3. Pause and present the commit command to the user before proceeding to Phase 6.

---

### Phase 6: Frontend Browse UI Pages, Search & Filtering (Next.js 16)

#### Execution Context
- Scope: Execute Phase 6 only. Create responsive browse views for Archetypes, Spells, Feats, Items, and Lineages with search, filters, infinite scroll, loading skeletons, and empty states.

#### Implementation Steps
1. Create shared encyclopedia layout `dungeoneer-frontend/app/encyclopedia/layout.tsx`:
   - Header with breadcrumb navigation back to `/characters`.
   - Tab navigation for quick jumping between reference categories.
    - Footer with the SRD CC-BY 4.0 attribution (pt-BR text), as required by the license.
2. Create reusable browse components in `dungeoneer-frontend/components/encyclopedia/`:
   - `SearchBar`: Debounced input (300ms) with search icon and clear button. Placeholder in pt-BR (e.g., "Buscar por nome ou descrição...").
   - `FilterBar`: Dropdown selectors for Ruleset (`SRD 2014`, `SRD 2024`, `Todos`), spell schools/levels, item types/rarities.
   - `InfiniteScrollSentinel`: Uses `IntersectionObserver` to trigger `fetchNextPage()` when scrolling into view, or renders a "Carregar Mais" button.
   - `ReferenceCard`: Dark Fantasy card styled with item details, badges, and expandable description text.
   - `ReferenceCardSkeleton`: Loading placeholder animation.
   - `EmptySearchResults`: State displayed when no records match search or filters ("Nenhum resultado encontrado").
3. Create browse pages:
   - `dungeoneer-frontend/app/encyclopedia/classes/page.tsx`: Archetypes browse view.
   - `dungeoneer-frontend/app/encyclopedia/spells/page.tsx`: Spells browse view with level & school filters.
   - `dungeoneer-frontend/app/encyclopedia/feats/page.tsx`: Feats browse view.
   - `dungeoneer-frontend/app/encyclopedia/items/page.tsx`: Items browse view with type & rarity filters.
   - `dungeoneer-frontend/app/encyclopedia/lineages/page.tsx`: Lineages browse view.
4. Write unit and component tests in `dungeoneer-frontend/__tests__/encyclopedia-browse.test.tsx`:
   - Test rendering of items list from mock data.
   - Test debounced search interaction triggering query update.
   - Test filter selection updating parameters.
   - Test empty state display when item list is empty.

#### Quality Gates & Verification
- Run frontend unit tests and production build:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/encyclopedia/ \
          dungeoneer-frontend/components/encyclopedia/ \
          dungeoneer-frontend/__tests__/encyclopedia-browse.test.tsx
  git commit -m "feat(frontend): add encyclopedia browse pages with search, filters and infinite scroll"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 6 section to `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`):
  1. UI pages and components introduced.
  2. Component and build test logs.
  3. Pause and present the commit command to the user before proceeding to Phase 7.

---

### Phase 7: Full-Stack Verification, Documentation & Final Walkthrough

#### Execution Context
- Scope: Execute Phase 7 only. Run complete end-to-end verification across backend and frontend, finalize delta specifications, check off all tasks in `tasks.md`, and complete the consolidated walkthrough.

#### Implementation Steps
1. Full-Stack Quality Verification:
   - Run backend tests and verify JaCoCo coverage:
     ```bash
     mvn -f player-character-service/backend/pom.xml clean verify
     ```
   - Run frontend tests and production build:
     ```bash
     npm --prefix dungeoneer-frontend run test:unit
     npm --prefix dungeoneer-frontend run build
     ```
2. Update Spec Tasks:
   - Mark all tasks in `openspec/changes/add-encyclopedia-browse/tasks.md` as completed (`[x]`).
   - Review delta specs to ensure all schema and endpoint changes are fully documented for future archival.
3. Finalize Cumulative Walkthrough:
   - Consolidate all execution summaries in `openspec/changes/add-encyclopedia-browse/walkthrough.md` in Portuguese (`pt-BR`).
   - Include actionable manual verification steps (navigating from main dashboard to each encyclopedia category, testing infinite scrolling across 50+ records, typing full-text search terms, applying rarity/level filters).
   - Document readiness to run `sdd-spec-archive`.

#### Quality Gates & Verification
- Verify all checks pass across the repository.
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/add-encyclopedia-browse/tasks.md \
          openspec/changes/add-encyclopedia-browse/walkthrough.md
  git commit -m "docs(spec): finalize tasks and cumulative walkthrough for add-encyclopedia-browse"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Deliver final summary and indicate readiness for archival via `sdd-spec-archive`.
