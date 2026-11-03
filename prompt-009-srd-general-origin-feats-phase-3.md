Focus: Backend / Spring Boot 4 (Java 21) / Application Validation & REST Controllers
Task: Execute Phase 3: Character Creation & Level-Up Validation

## Execution Context
- Active Plan: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/tasks.md`
- Linked Proposal: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/proposal.md`
- Domain Delta: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/specs/player-character-domain-delta.md`
- Scope: Execute Phase 3 only (Tasks 3.1 to 3.3). Verify prerequisite validation during character creation (`CreateCharacterService`) and level-up (`LevelUpService`) against newly seeded feats and dynamic prerequisite rules, and verify reference endpoints in `ReferenceDataControllerTest.java` for `/api/reference/feats` with `featTypes=GENERAL` and `featTypes=ORIGIN`. Do not proceed to subsequent phases.

## Implementation Steps
1. **Verify Character Creation Feat Prerequisite Validation:**
   - Inspect `CreateCharacterService` and ensure feat selections validate prerequisite rules (`prerequisiteRules`, minimum ability scores, and ruleset match).
   - Ensure selections failing prerequisites are rejected with a 400 Bad Request ProblemDetail.
   - Add or update unit tests in `CreateCharacterServiceTest` covering both compliant selections and prerequisite violation edge cases.
2. **Verify Level-Up Feat Prerequisite Validation:**
   - Inspect `LevelUpService` and ensure feat selection during ASI levels evaluates minimum ability scores, level, and prerequisite rules for newly seeded general feats.
   - Ensure invalid selections throw domain validation exceptions mapping to 400 Bad Request ProblemDetail.
   - Add or update unit tests in `LevelUpServiceTest` validating prerequisite evaluations.
3. **Reference Data Controller Integration Testing:**
   - Update `ReferenceDataControllerTest.java` to test `GET /api/reference/feats` with `featTypes=GENERAL` and `featTypes=ORIGIN`.
   - Verify that response payloads accurately return the expanded catalogs for both `SRD_2014` and `SRD_2024` with deserialized `mechanicalEffects` (including `spellChoice` for *Magic Initiate* / *Iniciado em Magia*).

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(feats): validate prerequisites in character creation and level up"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/009-seed-complete-srd-general-and-origin-feats/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 4 (Frontend Catalog & UI Verification).
Pause and prompt the user to commit progress before proceeding to Phase 4.
