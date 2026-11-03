Focus: Full-Stack / Specification & Review Gate
Task: Execute Phase 5: Delta Specs Consolidation & Final Review Gate

## Execution Context
- Active Plan: `openspec/changes/add-long-short-rest/tasks.md`
- Linked Proposal: `openspec/changes/add-long-short-rest/proposal.md`
- Scope: Execute Phase 5 only. Finalize change validation and documentation.

## Implementation Steps
1. Populate and finalize delta specs under `openspec/changes/add-long-short-rest/specs/`:
   - `specs/player-character-domain-delta.md`: Document short/long rest mechanics, RulesEngine delegation, temporary HP preservation, and the 0 HP non-resurrection invariant.
   - `specs/player-character-data-model-delta.md`: Document Flyway V27 migration changes, `exhaustion` (0–6) column on `characters`, and `hit_dice_spent` on `class_progressions`.
   - `specs/frontend-web-app-delta.md`: Document RestModal component, header rest action triggers, Hit Dice counter, and exhaustion indicator badge.
2. Execute full regression quality gates across both backend and frontend:
   - Backend verification: `mvn clean verify` (ensure ≥80% JaCoCo coverage on domain and application layers, and all unit/integration tests pass).
   - Frontend validation: `npm test` and build check in `dungeoneer-frontend/`.
3. Verify all acceptance criteria from `openspec/changes/add-long-short-rest/proposal.md` against the completed implementation.
4. Update `openspec/changes/add-long-short-rest/tasks.md`:
   - Mark all implementation tasks, test tasks, documentation tasks, and review gate items as completed `[x]`.
5. Finalize the cumulative walkthrough artifact `openspec/changes/add-long-short-rest/walkthrough.md` in Portuguese (pt-BR):
   - Consolidate all created and modified files across backend and frontend.
   - Summarize test execution logs and coverage validation results.
   - Provide manual UI smoke test instructions and user verification steps preparing the change for archival via `sdd-spec-archive`.

## Quality Gates & Verification
- Run local validation: `mvn clean verify` && `npm test`
- Commit locally: `git commit -m "docs(spec): finalize delta specs and review gate for add-long-short-rest"`

## Phase Handoff & Cumulative Walkthrough
Update or create a shared walkthrough artifact (`openspec/changes/add-long-short-rest/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase [N+1].
*(If this is the final phase, complete the walkthrough with a full consolidated summary of all changes and actionable next steps, such as manual UI/CLI validations or user review checklists).*
