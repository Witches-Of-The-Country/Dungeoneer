Focus: Frontend / Next.js 16 (React 19) / TailwindCSS 4 / Vitest
Task: Execute Phase 2: Frontend Level Up Wizard UI & State

## Execution Context
- Active Plan: `openspec/changes/level-up-half-feat-ability-choice/tasks.md`
- Linked Proposal: `openspec/changes/level-up-half-feat-ability-choice/proposal.md`
- Frontend Spec Delta: `openspec/changes/level-up-half-feat-ability-choice/specs/frontend-web-app-delta.md`
- Cumulative Walkthrough: `openspec/changes/level-up-half-feat-ability-choice/walkthrough.md`
- Scope: Execute Phase 2 only (Tasks 2.1 to 2.6). Implement half-feat ability selection, validation gating, summary display, mutation payload integration in `LevelUpWizardModal`, and unit tests. Do not proceed to Phase 3.

## Implementation Steps
1. **Half-Feat Detection & Interactive Selector (Step 3 - Feat Selection):**
   - In `level-up-wizard-modal.tsx`, detect whether the selected feat is a half-feat granting an attribute bonus choice (`abilityBonusChoice`) or static bonuses (`abilityBonuses`).
   - Render an interactive ability score selector sub-panel when a half-feat requiring a choice is selected. Restrict options to `allowedAbilities` (translated using `ABILITY_LABELS` or pt-BR domain labels).
   - Display the character's current score and projected score (+1, max cap of 20) for candidate attributes.
2. **Step Advancement Gating & Validation Feedback:**
   - In `handleAdvanceStep`, block advancing from Step 3 if a half-feat with `abilityBonusChoice` has no ability selected.
   - Display an informative error toast notifying the user to select an attribute for the feat bonus.
3. **Summary Confirmation & Mutation Payload Integration (Step 4 - Review & Submit):**
   - Update Step 4 summary to display the chosen ability boost alongside the feat name.
   - Pass the selected ability bonus as `abilityScoreIncreases: { [chosenAbility]: 1 }` in the level-up mutation payload alongside `selectedFeatId`.
4. **Unit Tests & Quality Gates:**
   - Update `level-up-wizard.test.tsx` to verify:
     - Half-feat ability selector renders the available choices when a half-feat is selected.
     - Advancement from Step 3 is prevented when an ability is not selected for a half-feat.
     - Selected ability reflects in Step 4 summary and is dispatched in the mutation payload.

## Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/
  git commit -m "feat(wizard): add half-feat ability score choice and validation on level up"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/level-up-half-feat-ability-choice/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test execution results.
3. Handoff context prepared for Phase 3 (Verification & Integration).
Pause and prompt the user to commit progress before proceeding to Phase 3.
