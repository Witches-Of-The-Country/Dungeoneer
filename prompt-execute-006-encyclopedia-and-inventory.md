# Phased Execution Prompt: Encyclopedia Modals, Homebrew Updates, and Inventory Store

Focus: Full-Stack / Spring Boot 4 (Java 21) / Clean Architecture / Next.js 16 (React 19 / TypeScript / Tailwind CSS 4)
Task: Execute Full Phased Implementation of Change `006-encyclopedia-modals-and-inventory-store`

## Global Execution Directives
- **Active Change Plan:** `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- **Approved Proposal:** `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- **Delta Specs:**
  - `openspec/changes/006-encyclopedia-modals-and-inventory-store/specs/player-character-domain-delta.md`
  - `openspec/changes/006-encyclopedia-modals-and-inventory-store/specs/frontend-web-app-delta.md`
- **Cumulative Walkthrough Artifact:** `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` (maintained strictly in Portuguese `pt-BR`).
- **Architectural & Stack Invariants:**
  - **Clean Architecture (Hexagonal):** Zero framework imports in backend `domain/model/` (no Spring, JPA, Hibernate, or Jackson annotations). DTOs with Jakarta Validation reside strictly in `application/dto/`. Web boundary controllers and mappers reside in `adapter/in/web/`.
  - **Ruleset Immutability Guard:** Official SRD content (`SRD_2014`, `SRD_2024`) must NEVER be editable. All update endpoints must verify `ruleset == Ruleset.HOMEBREW` and reject any other ruleset with RFC 7807 `400 Bad Request` or `403 Forbidden`.
  - **Build & Test Tools:** ALWAYS use global `mvn` (e.g., `mvn -f player-character-service/backend/pom.xml ...`), NEVER maven wrapper (`mvnw`). Maintain line coverage `>= 80%` on `domain` and `application` packages. Use `npm --prefix dungeoneer-frontend` for frontend tasks.
  - **Phase Pauses & Commit Commands:** Execute one phase at a time. After verifying each phase locally, provide the exact Conventional Commit command, update the cumulative walkthrough in `pt-BR`, mark completed tasks as `[x]` in `tasks.md`, and pause for user confirmation before proceeding to the next phase.

---

### Critical Path Overview
1. **Phase 1 (Backend Domain & Hydration Fixes):** Preserve `currencyPouch` and `avatarUrl` in `PlayerCharacter` immutability methods; integrate `CharacterSheetHydrator` into queries; enrich summary with subclass/multiclass progressions.
2. **Phase 2 (Backend Homebrew Updates & Security):** Add `PUT /api/reference/{category}/{id}` endpoints in `ReferenceDataController` and `HomebrewService`; enforce strict ruleset validation.
3. **Phase 3 (Frontend Currency Input Sizing):** Fix layout and number clipping in `create-homebrew-dialog.tsx`.
4. **Phase 4 (Frontend Sheet Feats Tab & Card Details):** Render hydrated feats with origin badges and detail modal; verify wallet persistence; display multiclass on character cards.
5. **Phase 5 (Frontend Encyclopedia Detail Modals):** Implement inspection dialogs for Spells, Feats, Items, and Lineages with "Editar" trigger for Homebrew.
6. **Phase 6 (Frontend Inventory Catalog & Store):** Create `ItemCatalogModal` with search, category filters, direct loot addition, and coin deduction checkout.
7. **Phase 7 (Verification & Review Gate):** Execute end-to-end regression suites, verify coverage, and consolidate final walkthrough.

---

### Phase 1: Backend Domain & Hydration Bug Fixes

Focus: Backend / Java 21 / Domain Modeling & Application Hydration
Task: Execute Phase 1: Fix Domain Immutability Wallet Resets, Character Sheet Hydration, and Multiclass Summary

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Scope: Execute Phase 1 only (Tasks 1.1 to 1.6). Fix `PlayerCharacter` state retention, ensure query hydration fills feat/item metadata, and expose subclass/multiclass in summaries. Do not proceed to homebrew update endpoints or frontend components.

#### Implementation Steps
1. **Fix `PlayerCharacter.java` Immutability Methods:**
   - In `withItems(List<CharacterItem> newItems)`, ensure `this.currencyPouch` and `this.avatarUrl` are passed to the copy constructor/factory instead of default empty values.
   - In `withFeats(List<CharacterFeat> newFeats)`, preserve `this.currencyPouch` and `this.avatarUrl`.
   - In `withLevelUp(...)`, preserve `this.currencyPouch` and `this.avatarUrl`.
2. **Inject & Apply `CharacterSheetHydrator`:**
   - In `CharacterQueryService.getCharacterSheet(UUID characterId)` and `UpdateCharacterService.updateCharacter(...)`, inject `CharacterSheetHydrator` and invoke `sheetHydrator.hydrate(character)` before assembling `CharacterSheetResult`, replacing static mappings with empty reference maps.
3. **Expose Feat Source in DTO:**
   - Add `source` field to `CharacterFeatDto` in `application/dto/`.
   - In `CharacterSheetHydrator`, extract or map the feat origin (`choicesJson` or Feat definition) and populate `source`.
4. **Enrich Multiclass & Subclass Character Summary:**
   - Update `CharacterSummaryResult`, `CharacterSummaryResponse`, `PlayerCharacterWebMapper`, and `CharacterQueryService.listCharacters` to include `subclassName` and the complete class progression list (`classes` containing class name and level).
5. **Unit & Application Tests:**
   - In `PlayerCharacterTest`, verify `currencyPouch` and `avatarUrl` are retained after calling `withItems`, `withFeats`, and `withLevelUp`.
   - In `CharacterQueryServiceTest`, verify that sheet retrieval returns hydrated feat and item definitions, and list retrieval includes subclass and multiclass breakdown.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=PlayerCharacterTest,CharacterQueryServiceTest,UpdateCharacterServiceTest
  ```
- Commit locally:
  ```bash
  git commit -m "fix(character): preserve currency pouch in domain mutations and hydrate sheet metadata"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update or create `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR recording:
1. List of files modified in Phase 1.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 2.

---

### Phase 2: Backend Homebrew Update Endpoints & Security Checks

Focus: Backend / REST API / Security Validation / Reference Data
Task: Execute Phase 2: Implement Homebrew Entity Update Use Cases and REST Endpoints with Ruleset Immutability Guard

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Scope: Execute Phase 2 only (Tasks 2.1 to 2.5). Add update operations for homebrew entities and enforce strict non-homebrew modification guards. Do not proceed to frontend changes.

#### Implementation Steps
1. **Declare Update Port in `CreateHomebrewUseCase`:**
   - Declare update methods: `updateSpell`, `updateFeat`, `updateItem`, `updateArchetype`, `updateLineage`, and `updateSubclass`.
2. **Implement Updates in `HomebrewService` with Ruleset Guard:**
   - Implement update methods in `HomebrewService`.
   - Load the existing entity by ID.
   - Enforce security validation: verify `entity.getRuleset() == Ruleset.HOMEBREW`. If the target entity has `SRD_2014` or `SRD_2024`, throw a `CharacterValidationException` or `IllegalStateException` preventing modification.
   - Update entity attributes and save through outbound repository ports.
3. **Expose `PUT /api/reference/{category}/{id}` Endpoints:**
   - In `ReferenceDataController`, add `PUT` endpoints for `/spells/{id}`, `/feats/{id}`, `/items/{id}`, `/classes/{id}`, `/lineages/{id}`, and `/subclasses/{id}`.
   - Annotate incoming request DTOs with `@Valid` and appropriate Jakarta Validation constraints.
4. **Unit & Web Layer Tests:**
   - In `HomebrewServiceTest`, verify successful modification of `HOMEBREW` entities and rejection of `SRD_2014`/`SRD_2024` entities.
   - In `ReferenceDataControllerTest`, verify that valid PUT requests return `200 OK` and attempted modifications of official reference entries return RFC 7807 error status (`400 Bad Request` or `403 Forbidden`).

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=HomebrewServiceTest,ReferenceDataControllerTest
  ```
- Commit locally:
  ```bash
  git commit -m "feat(reference): add homebrew update endpoints with srd immutability guard"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 2 progress to `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR:
1. List of files created/modified in Phase 2.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 3.

---

### Phase 3: Frontend Homebrew Currency Input Sizing Fix

Focus: Frontend / UI / Tailwind CSS 4 / Dialog Usability
Task: Execute Phase 3: Refactor Homebrew Item Currency Inputs for Multi-Digit Legibility

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Scope: Execute Phase 3 only (Tasks 3.1 to 3.2). Resolve clipping and spinner interference on currency inputs inside `create-homebrew-dialog.tsx`.

#### Implementation Steps
1. **Refactor Currency Cost Inputs in `create-homebrew-dialog.tsx`:**
   - Restructure the currency inputs (copper, silver, electrum, gold, platinum) into a clean, dedicated responsive grid row.
   - Apply utility classes `[appearance:textfield] [&::-webkit-outer-spin-button]:appearance-none [&::-webkit-inner-spin-button]:appearance-none` to suppress browser spin controls that obscure digits.
   - Ensure input fields have adequate width and padding so values like `50` or `1000` remain completely legible without truncation.
2. **Visual & Unit Validation:**
   - Verify typing two-digit and three-digit numbers displays correctly without text cropping or wrapping.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "fix(ui): expand homebrew currency input width and hide native spin buttons"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 3 progress to `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR:
1. List of files modified in Phase 3.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 4.

---

### Phase 4: Frontend Character Sheet Feats Tab & Modals

Focus: Frontend / React 19 / Character Sheet / Modal Inspection
Task: Execute Phase 4: Enhance Character Sheet Feats Tab with Origin Badges, Detail Inspection, and Multiclass Card Summary

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Scope: Execute Phase 4 only (Tasks 4.1 to 4.4). Fix the generic "Talento" cards in the character sheet, enable feat detail modal, confirm currency persistence in wallet, and render subclass/multiclass in `/characters`.

#### Implementation Steps
1. **Enhance `feats-tab.tsx`:**
   - Render the hydrated feat name, origin badge (e.g. Linhagem, Classe/Arquétipo, Antecedente, Personalizado), and description summary instead of generic placeholders.
   - Wire click interactions on feat cards to open a detail modal displaying full mechanical benefits and prerequisites.
2. **Validate Currency Persistence in `currency-pouch-card.tsx`:**
   - Ensure the wallet card correctly reflects persisted coins returned from the backend after sheet mutation without resetting to zero.
3. **Display Subclass & Multiclass on Character Cards:**
   - In `app/characters/page.tsx`, update character summary cards to render primary class with subclass (e.g., "Guerreiro (Mestre da Batalha)") and secondary multiclass levels (e.g., "/ Mago 2").
4. **Unit Tests:**
   - Add/update tests in `dungeoneer-frontend` for `feats-tab.tsx` ensuring hydrated feat details and origin badges render correctly.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "feat(character-sheet): render hydrated feats with origin badges and detail modal"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 4 progress to `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR:
1. List of files modified in Phase 4.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 5.

---

### Phase 5: Frontend Encyclopedia Detail Modals & Homebrew Edit

Focus: Frontend / Dark Fantasy UI / Encyclopedia / Homebrew Edit Flow
Task: Execute Phase 5: Build Detail Modals for Spells, Feats, Items, and Lineages with Homebrew Edit Support

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Delta Spec: `openspec/changes/006-encyclopedia-modals-and-inventory-store/specs/frontend-web-app-delta.md`
- Scope: Execute Phase 5 only (Tasks 5.1 to 5.6). Implement all 4 inspection modals, bind them to card clicks in Encyclopedia views, and add homebrew edit flow.

#### Implementation Steps
1. **Create Detail Modals:**
   - `SpellDetailModal.tsx`: display school, casting time, range, components, duration, damage rolls, and upcast scaling.
   - `FeatDetailModal.tsx`: display prerequisites, stat bonuses, ruleset source, and mechanical effects.
   - `ItemDetailModal.tsx`: display category, rarity, weight, cost, attunement, properties, and damage/armor mechanics.
   - `LineageDetailModal.tsx`: display traits, speed, size, and racial abilities.
2. **Wire Card Click Handlers in Encyclopedia Pages:**
   - In `app/encyclopedia/spells/page.tsx`, `feats/page.tsx`, `items/page.tsx`, and `lineages/page.tsx`, update `ReferenceCard` interactions to open the corresponding detail modal on card click.
3. **Add Homebrew Edit Integration:**
   - Inside detail modals, check if `ruleset === 'HOMEBREW'`.
   - If homebrew, display an "Editar" action button opening `create-homebrew-dialog.tsx` pre-filled in edit mode.
   - On submit, call `PUT /api/reference/{category}/{id}` and invalidate relevant TanStack Query keys.
   - For official SRD entries, hide the "Editar" button completely.
4. **Unit Tests:**
   - Add unit tests for `SpellDetailModal`, `ItemDetailModal`, and `FeatDetailModal` verifying rendering of mechanics and conditional visibility of the "Editar" button based on ruleset.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "feat(encyclopedia): introduce detail modals and edit action for homebrew entities"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 5 progress to `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR:
1. List of files created/modified in Phase 5.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 6.

---

### Phase 6: Frontend Inventory Item Catalog & Shopping Cart Modal

Focus: Frontend / Character Sheet / Inventory Management / Store Transactions
Task: Execute Phase 6: Build Rich Item Catalog Modal with Direct Add and Shopping Cart Purchase

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Proposal Reference: `openspec/changes/006-encyclopedia-modals-and-inventory-store/proposal.md`
- Scope: Execute Phase 6 only (Tasks 6.1 to 6.4). Replace the `<select>` item picker with `ItemCatalogModal` supporting search, filtering, free loot addition, and coin purchase checkout.

#### Implementation Steps
1. **Create `item-catalog-modal.tsx`:**
   - Implement catalog browser with search input, category tabs (Armas, Armaduras, Consumíveis, Itens Gerais, etc.), and rarity filters.
   - Provide item detail drawer/preview within the catalog dialog.
2. **Implement Dual Checkout Modes:**
   - **Adicionar Direto (Free Loot):** Add selected item with quantity selector directly to character inventory without currency deduction (`purchaseWithCurrency: false`).
   - **Loja / Carrinho (Store Purchase):** Show item unit and total cost in coins. Check character coin balance. If affordable, submit checkout with `purchaseWithCurrency: true` to trigger backend `CurrencyPouch.deductWithConversion`. If unaffordable, disable checkout with clear feedback.
3. **Integrate with `inventory-tab.tsx`:**
   - Replace the legacy dropdown select with a button triggering `ItemCatalogModal`.
   - Invalidate character sheet TanStack Query cache upon successful item addition.
4. **Unit Tests:**
   - Add unit tests for `ItemCatalogModal` verifying search filtering, category tab selection, quantity calculation, and mutation calls for both direct addition and store purchase.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "feat(inventory): add item catalog modal with direct addition and store purchase"
  ```

#### Phase Handoff & Cumulative Walkthrough
Append Phase 6 progress to `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR:
1. List of files created/modified in Phase 6.
2. Verification commands and test execution output.
3. Handoff context prepared for Phase 7.

---

### Phase 7: Verification, Review Gate & Final Walkthrough

Focus: Full-Stack Quality Assurance / SDD Spec Finalization
Task: Execute Phase 7: Run End-to-End Regression Suites, Audit Review Gate, and Consolidate Walkthrough

#### Execution Context
- Active Plan: `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md`
- Scope: Execute Phase 7 only (Tasks 7.1 to 7.3 and Review Gate). Verify backend and frontend test suites, ensure line coverage meets `>= 80%`, verify build artifacts, and mark tasks complete.

#### Implementation Steps
1. **Full Verification:**
   - Run complete test suite for `player-character-service` backend:
     ```bash
     mvn -f player-character-service/backend/pom.xml clean test
     ```
   - Verify JaCoCo line coverage meets `>= 80%` on modified domain and application services.
   - Run frontend unit test suite and production build:
     ```bash
     npm --prefix dungeoneer-frontend run test:unit
     npm --prefix dungeoneer-frontend run build
     ```
2. **Audit Review Gate Checklist:**
   - Verify all 8 acceptance criteria from `proposal.md` are fulfilled.
   - Ensure zero regressions in character creation, level up, and dice roll flows.
   - Ensure all tasks in `openspec/changes/006-encyclopedia-modals-and-inventory-store/tasks.md` are marked `[x]`.
3. **Consolidated Walkthrough Summary:**
   - Finalize `openspec/changes/006-encyclopedia-modals-and-inventory-store/walkthrough.md` in pt-BR with:
     - Consolidated summary of all changes across Phases 1–6.
     - Full test results, test coverage report, and build metrics.
     - Manual UI verification checklist (opening encyclopedia modals, editing homebrew, checking wallet retention, testing catalog cart).
     - Readiness confirmation for archiving via `sdd-spec-archive`.

#### Quality Gates & Verification
- Run validation commands:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  npm --prefix dungeoneer-frontend run build
  ```
- Commit locally:
  ```bash
  git commit -m "docs(spec): mark change 006 tasks complete and record consolidated walkthrough"
  ```
