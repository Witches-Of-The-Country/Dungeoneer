# Phased Execution Prompt: Refactor Wizard HP, Full SRD Translations, Multiclassing and Character Sheet Enhancements

Focus: Full-Stack / Spring Boot 4 (Java 21) / Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Execute Full Phased Implementation of Change `refactor-wizard-hp-srd-and-session`

## Global Execution Directives
- **Active Change Plan:** `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md`
- **Approved Proposal:** `openspec/changes/refactor-wizard-hp-srd-and-session/proposal.md`
- **Delta Specs:**
  - `openspec/changes/refactor-wizard-hp-srd-and-session/specs/player-character-data-model-delta.md`
  - `openspec/changes/refactor-wizard-hp-srd-and-session/specs/player-character-domain-delta.md`
  - `openspec/changes/refactor-wizard-hp-srd-and-session/specs/frontend-web-app-delta.md`
- **Cumulative Walkthrough Artifact:** `openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` (maintained strictly in Portuguese `pt-BR`).
- **Architectural Invariants:**
  - Zero framework imports in backend `domain/model/` (no Spring, JPA, or Lombok).
  - Clean Architecture layer flow: `domain` <- `application` <- `adapter`.
  - REST errors mapped via RFC 7807 `ProblemDetail` responses.
  - Multi-tenant / User isolation: character notes and creation strictly verified against user identity.
  - Sequential Phase Gates: execute one phase at a time. Run local verification, present the exact Git commit command, update the cumulative walkthrough, and pause for user approval before moving to the next phase.

---

### Phase 1: Backend Domain, Flyway V6 Migration, Note Update & Multiclass Creation Support

Focus: Backend / Spring Boot 4 (Java 21) / Clean Architecture / PostgreSQL Flyway
Task: Execute Phase 1: Backend Domain, Migration V6, Note Update Port/Adapter and Multiclass Creation

#### Execution Context
- Active Plan: `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md` (Sections 2 and 3)
- Delta Specs: `specs/player-character-data-model-delta.md` and `specs/player-character-domain-delta.md`
- Scope: Execute Phase 1 only. Implement Flyway V6 migration, domain model adjustments, note update use case, multiclass/skills persistence in character creation, REST endpoints, and automated tests. Do not proceed to frontend phases.

#### Implementation Steps
1. **Flyway Migration V6 (`player-character-service`):**
   - Create migration `V6__add_skill_proficiencies_to_characters.sql` under `src/main/resources/db/migration/`.
   - Add column `skill_proficiencies VARCHAR[] NOT NULL DEFAULT '{}'` to table `player_character.characters`.
2. **Domain Model & Note Update Use Case:**
   - Update `PlayerCharacter` domain model to hold and expose `List<String> skillProficiencies`.
   - In `CharacterNote` domain entity, implement business mutation methods for updating title and body, enforcing invariants (`title` not blank and <= 255 chars; `body` not blank and <= 10,000 chars; throw domain exception on violation).
   - In `application/port/in/`, create inbound port `UpdateCharacterNoteUseCase` and application DTO `UpdateCharacterNoteCommand`.
   - In `CharacterNoteService`, implement `updateNote` validating character existence and ownership (verifying `userId` and `characterId`).
3. **Multiclass & Skill Proficiencies in Character Creation:**
   - Extend `CreateCharacterCommand` and `CreateCharacterRequest` to accept `List<ClassProgressionCommand> classProgressions` and `List<String> skillProficiencies`.
   - In `CreateCharacterUseCase` implementation, validate that when class progressions are supplied:
     - The sum of class levels across all progressions equals the total character level.
     - Save all class progression records linked to the created character.
     - Persist `skillProficiencies` into the character entity.
4. **Web Adapter & REST Endpoint:**
   - In `adapter/in/web/`, define `UpdateCharacterNoteRequest` DTO with Jakarta validation annotations (`@NotBlank`, `@Size`).
   - In `PlayerCharacterController`, expose endpoint `PUT /api/characters/{id}/notes/{noteId}` returning `200 OK` with `CharacterNoteResponse`.
   - Update `PlayerCharacterWebMapper` to support the note update and creation DTO mappings.
5. **Automated Backend Testing:**
   - Unit tests: verify `CharacterNote` update invariants, ownership checks in `CharacterNoteService`, and multiclass level sum validation in character creation.
   - MockMvc integration tests: verify `PUT /api/characters/{id}/notes/{noteId}` for happy path (200), validation errors (400), and unauthorized/mismatched ownership (403/404).

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml clean test
  ```
- Commit locally:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/V6__add_skill_proficiencies_to_characters.sql \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/ \
          player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/
  git commit -m "feat(backend): add Flyway V6 skill_proficiencies, UpdateCharacterNoteUseCase, and multiclass creation support"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update or create the shared walkthrough artifact (`openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in Phase 1 (migration, domain entities, ports, service, controller, and test suites).
2. Verification and test execution results (`mvn clean test`).
3. Handoff context prepared for Phase 2 (Frontend Auth Session Guard, Translations & Sheet Enhancements).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 2.*

---

### Phase 2: Frontend Auth Session Guard, SRD Translations, NoteCard Inline Editing & Numeric HP Controls

Focus: Frontend / Next.js 16 / React 19 / TypeScript / TailwindCSS 4 / TanStack Query
Task: Execute Phase 2: Session Guard, SRD PT-BR Expansion, Note Inline Editing and Numeric HP Controls

#### Execution Context
- Active Plan: `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md` (Sections 3, 4, 5, 6)
- Delta Specs: `specs/frontend-web-app-delta.md`
- Scope: Execute Phase 2 only. Implement frontend authentication session verification, dictionary translations, inline note editing on `NoteCard`, and advanced HP controls on the character sheet. Do not start wizard refactoring yet.

#### Implementation Steps
1. **Auth Session Guard (`useAuth` & `client.ts`):**
   - In `lib/hooks/use-auth.ts`, add asynchronous session integrity verification on mount (`useEffect`). If stored token fails validation against backend or returns HTTP 401/404, trigger `clearAuth()` to clear stale session state and sync the navigation bar.
   - In `lib/api/client.ts`, ensure any HTTP 401 response interceptor automatically invokes `clearAuth()`.
2. **Comprehensive SRD Translations (`lib/srd-translations.ts` & `lib/api/characters.ts`):**
   - Expand translation dictionaries in `lib/srd-translations.ts` to cover racial traits (`traits_summary`), lineage and background descriptions, origin feats (`originFeatDescription`), and equipment proficiencies (armor, weapons, tools).
   - In `lib/api/characters.ts`, ensure TanStack Query transformers map all textual SRD fields cleanly into Portuguese (`pt-BR`) with transparent fallbacks.
3. **Inline Note Editing in `NoteCard`:**
   - In `lib/api/characters.ts`, add `updateCharacterNote` API function and `useUpdateCharacterNote` mutation hook with automatic query cache invalidation.
   - In `components/note-card.tsx`, add an edit action button (pencil icon). Clicking toggles inline editable input fields for title and body, accompanied by "Salvar" and "Cancelar" buttons.
4. **Advanced Numeric HP Controls in Character Sheet (`app/characters/[id]/page.tsx`):**
   - In the Hit Points card:
     - Retain Line 1 standard incremental buttons (`-1 PV` and `+1 PV`).
     - Add Line 2 dedicated numeric adjustments:
       - Left: `[-]` button followed by number input (`placeholder="Dano"`). Clicking `-` subtracts the value from current HP (clamped to minimum 0) and resets input to empty.
       - Right: Number input (`placeholder="Cura"`) followed by `[+]` button. Clicking `+` adds the value to current HP (clamped to `hitPointsMax`) and resets input to empty.
5. **Frontend Automated Unit Tests:**
   - Update and expand `__tests__/incremental-notes-and-translations.test.tsx` and `__tests__/character-sheet.test.tsx` to verify:
     - Inline note editing behavior and query invalidation.
     - Numeric HP damage/healing controls, limits (0 and max HP), and input auto-reset.
     - PT-BR translation coverage for racial traits and equipment proficiencies.
     - Session cleanup on 401/404 responses.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/lib/hooks/use-auth.ts \
          dungeoneer-frontend/lib/api/client.ts \
          dungeoneer-frontend/lib/api/characters.ts \
          dungeoneer-frontend/lib/srd-translations.ts \
          dungeoneer-frontend/components/note-card.tsx \
          dungeoneer-frontend/app/characters/\[id\]/page.tsx \
          dungeoneer-frontend/__tests__/
  git commit -m "feat(frontend): add session guard, full SRD translations, note inline editing, and numeric HP controls"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` in pt-BR):
1. Document modified files, UI components, translation keys, and tests updated in Phase 2.
2. Record unit test execution outputs.
3. Handoff context prepared for Phase 3 (Character Wizard UX Refactoring).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 3.*

---

### Phase 3: Character Wizard UX Refactoring (Level, Per-Class HP Roll Cache, Class Skills & Multiclass)

Focus: Frontend / Next.js 16 / React 19 / State Management / Character Creation Wizard
Task: Execute Phase 3: Wizard Step 3 Refactoring, HP Roll Cache, Skill Selection and Multiclass Distribution

#### Execution Context
- Active Plan: `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md` (Sections 1 and 2)
- Delta Specs: `specs/frontend-web-app-delta.md`
- Scope: Execute Phase 3 only. Reorganize wizard steps, relocate level and HP roll controls to Step 3, manage per-class HP roll cache, implement class skill selection, multiclass level distribution, and background skill collision detection.

#### Implementation Steps
1. **Step 1 ("Nome & Regras") Simplification:**
   - In `app/characters/new/page.tsx`, simplify Step 1 to only contain Character Name and Ruleset selector (`SRD_2014` vs `SRD_2024`). Remove level selection from Step 1.
2. **Step 3 ("Classe") Reorganization:**
   - Relocate the Total Initial Level selector (levels 1 to 20) to Step 3.
   - Relocate HP calculation method selection (`MAX`, `AVERAGE`, `ROLLED`) and dice rolling controls into Step 3.
   - **Per-Class HP Roll Cache (`hpRollsByClass`):** Store roll responses indexed by class identifier (`Record<string, HpRollResponse>`). When switching between classes, retain rolled dice and retrieve previous rolls when returning to a previously rolled class.
   - **Step 3 Transition Guard:** Prevent advancing from Step 3 to Step 4 if `level > 1` and `ROLLED` method is active but no roll has been performed for the active class.
3. **Class Skill Selection in Step 3:**
   - Catalog class skill options and permissible choice limits (e.g., Fighter: 2, Rogue: 4, Bard: 3) from reference rules.
   - Display eligible class skill options with interactive selection and real-time choice counter.
   - Store selected class skills in wizard state.
4. **Multiclass Level Distribution in Step 3 (Level > 1):**
   - When Total Level > 1, render multiclass distribution panel:
     - Primary class defaults to level 1.
     - "Adicionar Classe Secundária" action allowing selection of additional classes and allocation of remaining levels.
     - Enforce synchronous validation: sum of assigned levels must exactly equal Total Initial Level to unlock advancement to Step 4.
5. **Step 4 Background Skill Collision Handling & Wizard Submission:**
   - In Step 4 (Background), check for overlap between selected class skills and background skills. Display badge/alert if collision occurs and allow player to adjust choices.
   - In final submission (`createCharacter`), send consolidated `skillProficiencies` array and `classProgressions` payload matching backend contract.
6. **Frontend Wizard Unit Tests:**
   - Update `__tests__/wizard-ability-roll-guard.test.tsx` and `__tests__/character-wizard.test.tsx` to verify:
     - Step 1 only prompts for name and ruleset.
     - Step 3 handles initial level, class selection, per-class HP roll caching, and transition guards.
     - Multiclass level distribution enforces exact sum equality.
     - Class skill selection limits and collision warning in Step 4.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/app/characters/new/page.tsx \
          dungeoneer-frontend/lib/api/characters.ts \
          dungeoneer-frontend/__tests__/wizard-ability-roll-guard.test.tsx \
          dungeoneer-frontend/__tests__/character-wizard.test.tsx
  git commit -m "feat(frontend): refactor wizard with Step 3 level/HP, per-class roll cache, skills, and multiclass"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` in pt-BR):
1. Detailed record of wizard state changes, component refactorings, and test outcomes.
2. Verification test report.
3. Handoff context prepared for Phase 4 (Full Regression & Archival Readiness).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 4.*

---

### Phase 4: Full Regression Testing, Delta Specs Alignment, Final Walkthrough & Archive Gate

Focus: Full-Stack / Quality Assurance / Documentation & SDD Archival Readiness
Task: Execute Phase 4: Full Verification, Spec Alignment, Final Walkthrough and Archive Gate

#### Execution Context
- Active Plan: `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md`
- Linked Proposal: `openspec/changes/refactor-wizard-hp-srd-and-session/proposal.md`
- Scope: Execute Phase 4 only. Run full automated test suites across backend and frontend, verify production builds, ensure all delta specs and tasks are satisfied, and complete the cumulative walkthrough.

#### Implementation Steps
1. **Full Backend Verification:**
   - Run complete test suite and code coverage analysis:
     ```bash
     mvn -f player-character-service/backend/pom.xml clean verify
     ```
   - Ensure JaCoCo coverage meets or exceeds the 80% line coverage threshold across domain and application layers.
2. **Full Frontend Verification & Production Build:**
   - Run complete frontend unit tests:
     ```bash
     npm --prefix dungeoneer-frontend run test:unit
     ```
   - Run frontend production build check:
     ```bash
     npm --prefix dungeoneer-frontend run build
     ```
   - Verify no TypeScript or lint regressions exist.
3. **Spec Alignment & Tasks Completion:**
   - Review `openspec/changes/refactor-wizard-hp-srd-and-session/tasks.md` and mark all completed tasks with `[x]`.
   - Verify that delta specs in `openspec/changes/refactor-wizard-hp-srd-and-session/specs/` accurately reflect the final implementation.
4. **Consolidated Cumulative Walkthrough:**
   - Complete `openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` in Portuguese (`pt-BR`) with:
     - Consolidated summary of all changes implemented across backend and frontend.
     - Final test and build verification results.
     - Step-by-step manual validation checklist for the user (Wizard creation flow with multiclass/skills, sheet inline note edit, numeric HP damage/healing, and session cleanup after volume resets).
     - Confirmation of readiness to execute `sdd-spec-archive`.

#### Quality Gates & Verification
- Run complete test verification across workspace:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Commit locally:
  ```bash
  git add openspec/changes/refactor-wizard-hp-srd-and-session/
  git commit -m "docs(openspec): complete walkthrough and mark tasks done for refactor-wizard-hp-srd-and-session"
  ```

#### Phase Handoff & Final Walkthrough Completion
Ensure `openspec/changes/refactor-wizard-hp-srd-and-session/walkthrough.md` is fully consolidated. Inform the user in Portuguese (`pt-BR`) that all implementation phases and quality gates have passed, and that the change is ready for archival via `sdd-spec-archive`.
