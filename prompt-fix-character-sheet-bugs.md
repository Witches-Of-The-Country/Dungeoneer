Focus: Full-Stack / Spring Boot 4 (Java 21) / Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query / @dnd-kit)
Task: Execute Full Implementation of Change `fix-character-sheet-bugs` across Backend and Frontend

## Execution Context
- Active Plan: `openspec/changes/fix-character-sheet-bugs/tasks.md`
- Linked Proposal: `openspec/changes/fix-character-sheet-bugs/proposal.md`
- Delta Specs:
  - `openspec/changes/fix-character-sheet-bugs/specs/player-character-data-model-delta.md`
  - `openspec/changes/fix-character-sheet-bugs/specs/player-character-domain-delta.md`
  - `openspec/changes/fix-character-sheet-bugs/specs/frontend-web-app-delta.md`
- Core Objectives:
  1. Bug 1: Enforce HP roll guard requiring class selection both in the creation wizard (UI guard & tooltip) and in the backend `RollService` use case (`CharacterValidationException`).
  2. Bug 2: Map all SRD reference content (classes, races/lineages, backgrounds, spells, skills) to PT-BR via frontend lookup dictionaries in `lib/srd-translations.ts` with transparent fallback.
  3. Bug 3: Convert character notes from a single monolithic string to incremental, chronologically ordered records via Flyway V5 migration, Clean Architecture domain model, application use cases, JPA repository adapter, REST endpoints (`POST`/`GET /api/characters/{id}/notes`), and Next.js UI components (`NoteCard`, `AddNoteForm`).
  4. Bug 4: Eliminate drag-and-drop cursor latency in `DraggableScoreChip` by removing `transition-all` CSS during active dragging or leveraging `@dnd-kit/core` `<DragOverlay>`.
- Architectural & Quality Constraints:
  - Clean Architecture: Zero framework imports in backend `domain/model/`. No Spring, JPA, or Lombok on domain entities.
  - Error Handling: REST API errors must follow RFC 7807 `ProblemDetail` via `@ControllerAdvice`.
  - Language Separation: Backend code, DTOs, domain models, commit messages, and automated tests strictly in English. User-facing frontend UI strings and shared walkthrough strictly in Portuguese (`pt-BR`).
  - Phase Pauses: Follow the atomic phases sequentially. Validate quality gates locally, present ready-to-run Conventional Commit commands, update the cumulative walkthrough, and pause for approval between phases.

---

### Phase 1: Backend Domain Entities, Flyway V5 Migration & HP Roll Guard

#### Execution Context
- Scope: Execute Phase 1 only. Implement the database schema migration, domain entity invariants, outbound repository port, JPA persistence adapter, and the domain-level validation guard for hit point rolls.

#### Implementation Steps
1. Create Flyway migration script `V5__create_character_note_table.sql` in `player-character-service/backend/src/main/resources/db/migration/`:
   - Table `player_character.character_notes` (`id UUID PRIMARY KEY`, `character_id UUID NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE`, `title VARCHAR(255) NOT NULL`, `body TEXT NOT NULL`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`).
   - Index `idx_character_notes_character_id` on `player_character.character_notes(character_id)`.
   - Data migration: Migrate existing non-empty `notes` from `player_character.characters` into initial note records with title `'Nota Inicial'`.
2. Implement pure domain entity `CharacterNote` in `domain/model/`:
   - Attributes: `id` (UUID), `characterId` (UUID), `title` (String), `body` (String), `createdAt` (Instant).
   - Domain invariants: `title` non-null, non-blank, max 255 chars; `body` non-null, non-blank, max 10,000 chars. Violations must throw `CharacterValidationException`.
   - Deprecate the legacy `notes` property in `PlayerCharacter` domain model.
3. Define outbound repository port `CharacterNoteRepositoryPort` in `application/port/out/`:
   - `CharacterNote save(CharacterNote note)`
   - `List<CharacterNote> findAllByCharacterIdOrderByCreatedAtDesc(UUID characterId)`
4. Implement JPA persistence adapter in `adapter/out/persistence/`:
   - Entity `CharacterNoteJpaEntity` mapped to `player_character.character_notes`.
   - Spring Data interface `SpringDataCharacterNoteRepository`.
   - Adapter `CharacterNotePersistenceAdapter` implementing `CharacterNoteRepositoryPort`.
5. Guard HP roll in `RollService.java` (`application/service/`):
   - Explicitly validate that `archetypeId` in `RollHitPointsCommand` is non-null and refers to an existing archetype.
   - Throw `CharacterValidationException` if missing, mapping to RFC 7807 Bad Request / Unprocessable Entity.
6. Write unit tests:
   - `CharacterNoteTest`: Domain invariants (rejection of blank/null/oversized title and body).
   - `RollServiceTest`: Rejection of null or missing archetype on hit point roll.

#### Quality Gates & Verification
- Run backend unit and persistence tests:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit command:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/V5__create_character_note_table.sql \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterNote.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/CharacterNoteRepositoryPort.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNoteJpaEntity.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataCharacterNoteRepository.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapter.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RollService.java \
          player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/
  git commit -m "feat(backend): add CharacterNote domain model, V5 Flyway migration, and HP roll validation"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Create or update `openspec/changes/fix-character-sheet-bugs/walkthrough.md` in Portuguese (`pt-BR`):
  1. Detail modified/created backend domain and persistence files.
  2. Document test execution outputs and verification results.
  3. Pause and present the commit command to the user before proceeding to Phase 2.

---

### Phase 2: Backend Use Cases, REST Endpoints & API Integration Tests

#### Execution Context
- Scope: Execute Phase 2 only. Implement the inbound use case ports, application service, REST controller endpoints (`POST` and `GET` for character notes), DTO validation, and controller integration tests.

#### Implementation Steps
1. Create inbound use case ports and application DTOs:
   - Ports in `application/port/in/`: `AddCharacterNoteUseCase`, `ListCharacterNotesUseCase`.
   - Application DTOs in `application/dto/`: `AddCharacterNoteCommand` (record with `characterId`, `userId`, `title`, `body`), `CharacterNoteResult` (record with `id`, `characterId`, `title`, `body`, `createdAt`).
2. Implement `CharacterNoteService` in `application/service/`:
   - Implement `AddCharacterNoteUseCase` and `ListCharacterNotesUseCase`.
   - Validate character ownership / existence before saving or listing notes.
3. Implement Web Adapter components in `adapter/in/web/`:
   - DTOs: `CreateCharacterNoteRequest` with Jakarta `@NotBlank`, `@Size(max = 255)` on title, `@Size(max = 10000)` on body; `CharacterNoteResponse`.
   - Web Mapper updates in `PlayerCharacterWebMapper`: map request to command, result to response.
   - REST Controller endpoints in `PlayerCharacterController`:
     - `POST /api/characters/{id}/notes` (returns `201 Created` with `CharacterNoteResponse`).
     - `GET /api/characters/{id}/notes` (returns `200 OK` with `List<CharacterNoteResponse>`).
4. Write comprehensive tests in `player-character-service/backend/src/test/java/`:
   - `CharacterNoteServiceTest`: Unit tests for adding and listing notes, character ownership checks, empty list handling.
   - `PlayerCharacterControllerTest`: MockMvc integration tests verifying successful creation (201), chronological retrieval (200), validation errors (400 for empty title or oversized body), and HP roll rejection without class selection (400/422).

#### Quality Gates & Verification
- Run complete test suite and verify JaCoCo code coverage:
  ```bash
  mvn -f player-character-service/backend/pom.xml clean verify
  ```
- Commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/ \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteService.java \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/ \
          player-character-service/backend/src/test/java/
  git commit -m "feat(backend): add REST endpoints and use cases for incremental character notes"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/fix-character-sheet-bugs/walkthrough.md` in Portuguese (`pt-BR`):
  1. Summary of REST endpoints and application services introduced.
  2. MockMvc and JaCoCo coverage report confirmation (>= 80%).
  3. Pause and present the commit command to the user before proceeding to Phase 3.

---

### Phase 3: Frontend Wizard HP Roll Guard & Drag-and-Drop Optimization

#### Execution Context
- Scope: Execute Phase 3 only. Resolve Bug 1 and Bug 4 in the Next.js frontend creation wizard (`dungeoneer-frontend/app/characters/new/page.tsx`).

#### Implementation Steps
1. Resolve Bug 1 (HP Roll Guard before class selection):
   - In `dungeoneer-frontend/app/characters/new/page.tsx`, check whether a class is selected (`selectedClass != null` or `character.class != null`).
   - If no class is selected:
     - Disable the HP roll button (`disabled={!selectedClass}`).
     - Display a clear informative tooltip / banner message: `"Selecione uma classe primeiro"`.
   - Ensure the roll calculation passes the exact `hit_die` associated with the chosen class.
2. Resolve Bug 4 (Eliminate Drag-and-Drop Latency):
   - In `dungeoneer-frontend/app/characters/new/page.tsx` (around `DraggableScoreChip`):
     - Identify CSS class `transition-all` causing CSS animation interpolation on pointer coordinates.
     - Conditionally disable transition while actively dragging (e.g. `isDragging ? "" : "transition-all duration-200"`) or decouple with `@dnd-kit/core` `<DragOverlay>`.
     - Ensure instantaneous pointer tracking with sub-16ms latency during rapid mouse movements.
3. Update and run frontend tests:
   - In `dungeoneer-frontend/__tests__/wizard-ability-roll-guard.test.tsx`:
     - Test that the roll button is disabled and shows the tooltip message when no class is selected.
     - Test that selecting a class enables the roll button with the matching hit die.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit command:
  ```bash
  git add dungeoneer-frontend/app/characters/new/page.tsx \
          dungeoneer-frontend/__tests__/wizard-ability-roll-guard.test.tsx
  git commit -m "fix(frontend): guard wizard HP roll and eliminate drag-and-drop latency"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/fix-character-sheet-bugs/walkthrough.md` in Portuguese (`pt-BR`):
  1. Changes made to `new/page.tsx` for roll guard and drag responsiveness.
  2. Test results from `wizard-ability-roll-guard.test.tsx`.
  3. Pause and present the commit command to the user before proceeding to Phase 4.

---

### Phase 4: Frontend SRD Translations, Incremental Notes UI & End-to-End Verification

#### Execution Context
- Scope: Execute Phase 4 only. Implement SRD translations (Bug 2), build the incremental notes UI components and React Query integration (Bug 3), update character sheet pages, run all quality checks, and consolidate documentation.

#### Implementation Steps
1. Implement SRD Translation Maps (Bug 2):
   - Create `dungeoneer-frontend/lib/srd-translations.ts`:
     - Export dictionaries: `CLASS_NAMES_PT`, `RACE_NAMES_PT`, `BACKGROUND_NAMES_PT`, `SPELL_NAMES_PT`, `SKILL_NAMES_PT`.
     - Export helper `translateSrd(key: string, dictionary: Record<string, string>): string` returning mapped PT-BR string or original key as fallback.
   - Apply translations across TanStack Query `select` transformers and component rendering in:
     - `app/characters/page.tsx` (character list cards)
     - `app/characters/[id]/page.tsx` (character sheet view)
     - `app/characters/[id]/edit/page.tsx` (character edit view)
2. Implement Incremental Notes UI (Bug 3):
   - Update `dungeoneer-frontend/lib/api/characters.ts`:
     - Define `CharacterNote` TypeScript interface (`id`, `characterId`, `title`, `body`, `createdAt`).
     - Add `useCharacterNotes(characterId)` query hook.
     - Add `useAddCharacterNote(characterId)` mutation hook with query invalidation.
   - Create `dungeoneer-frontend/components/note-card.tsx`:
     - Render `title`, `body`, and formatted PT-BR timestamp (`toLocaleDateString('pt-BR')`).
   - Create `dungeoneer-frontend/components/add-note-form.tsx`:
     - Inputs for `title` (max 255) and `body` (max 10,000) with client validation and submission via mutation hook.
   - Refactor `app/characters/[id]/page.tsx` and `app/characters/[id]/edit/page.tsx`:
     - Replace legacy monolithic `notes` textarea with `<AddNoteForm>` and list of `<NoteCard>` in reverse chronological order.
3. Update and run frontend tests:
   - `dungeoneer-frontend/__tests__/character-sheet.test.tsx`: Verify multiple notes rendered, addition of new note, validation of required title and body.
   - `dungeoneer-frontend/__tests__/character-list-and-edit.test.tsx`: Verify PT-BR translation fallback and display.
   - Run full frontend test suite and production build check.
4. Verify delta specs & documentation updates:
   - Ensure all tasks in `openspec/changes/fix-character-sheet-bugs/tasks.md` are completed.
   - Prepare delta specs for future archival via `sdd-spec-archive`.

#### Quality Gates & Verification
- Run complete test suite and build across the repository:
  ```bash
  npm --prefix dungeoneer-frontend run build
  npm --prefix dungeoneer-frontend run test:unit
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit command:
  ```bash
  git add dungeoneer-frontend/lib/srd-translations.ts \
          dungeoneer-frontend/lib/api/characters.ts \
          dungeoneer-frontend/components/note-card.tsx \
          dungeoneer-frontend/components/add-note-form.tsx \
          dungeoneer-frontend/app/characters/ \
          dungeoneer-frontend/__tests__/ \
          openspec/changes/fix-character-sheet-bugs/
  git commit -m "feat(frontend): integrate SRD PT-BR translations and incremental notes UI"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Complete the shared walkthrough `openspec/changes/fix-character-sheet-bugs/walkthrough.md` in Portuguese (`pt-BR`):
  1. Consolidated summary of all 4 bug resolutions across backend and frontend.
  2. Final verification and build status.
  3. Actionable checklist for manual testing (testing DnD fluidity in browser, verifying PT-BR strings on sheet, creating notes, and verifying wizard HP button disabled state).
  4. Final prompt indicating readiness to execute `sdd-spec-archive`.
