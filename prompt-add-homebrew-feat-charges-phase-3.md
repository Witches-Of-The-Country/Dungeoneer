Focus: Frontend / Next.js 16 (React 19) / TailwindCSS 4 / Vitest
Task: Execute Phase 3: Frontend Homebrew Dialog, Feat Card Charges & Vitest Tests

## Execution Context
- Active Plan: `openspec/changes/add-homebrew-feat-charges/tasks.md`
- Linked Proposal: `openspec/changes/add-homebrew-feat-charges/proposal.md`
- Frontend Spec Delta: `openspec/changes/add-homebrew-feat-charges/specs/frontend-web-app-delta.md`
- Cumulative Walkthrough: `openspec/changes/add-homebrew-feat-charges/walkthrough.md`
- Scope: Execute Phase 3 only. Implement frontend types, charge builder form in `create-homebrew-dialog.tsx`, charge and recovery indicators in `feats-tab.tsx`, Vitest component tests (TP-04), and fill delta specifications. Complete final review gate.

## Implementation Steps
1. **Frontend Types & API Client:**
   - In `dungeoneer-frontend/lib/api/homebrew.ts` (and relevant type definitions), add `ChargeType`, `ChargeRecoveryType`, and feat charge configuration fields to homebrew payload interfaces and feat DTOs.
2. **Homebrew Feat Charge Builder UI:**
   - In `create-homebrew-dialog.tsx`, add an optional "Charges & Recovery" builder section for feats.
   - Support selecting max charges formula (`FIXED`, `ABILITY_MODIFIER`, `PROFICIENCY`, `LEVEL`), choosing the linked ability when `ABILITY_MODIFIER` is selected, and configuring independent short and long rest recovery rules (`NONE`, `ALL`, `FIXED`, `MODIFIER`, `PROFICIENCY`, `LEVEL`).
   - Default short rest recovery to `NONE` and long rest recovery to `ALL`. Validate required fields before submission.
3. **Character Sheet Feat Card Display:**
   - In `feats-tab.tsx`, display charge trackers on feat cards (e.g., current/max charges badges, recovery tags for short/long rest).
   - Maintain consistent Dark Fantasy RPG design tokens and Portuguese (pt-BR) UI labels.
4. **Component Tests & Spec Delta Documentation:**
   - Add or update Vitest unit tests covering:
     - TP-04: Homebrew dialog charge builder state management and correct payload submission.
     - Feats tab rendering of charges and recovery rules.
   - Fill delta specs under `openspec/changes/add-homebrew-feat-charges/specs/` (`player-character-domain-delta.md`, `player-character-data-model-delta.md`, `frontend-web-app-delta.md`) to reflect implemented behavior.
   - Check off completed tasks in `openspec/changes/add-homebrew-feat-charges/tasks.md`.

## Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git commit -m "feat(frontend): add feat charge builder in homebrew dialog and display on feats tab"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/add-homebrew-feat-charges/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Final consolidated summary of all phases, verifying that all acceptance criteria and tasks in `tasks.md` are fulfilled, with actionable verification next steps (e.g., manual smoke tests in the UI, readiness for `sdd-spec-archive`).
