Focus: Frontend / Next.js & React UI
Task: Execute Phase 4: Frontend UI Integration & Component Tests

## Execution Context
- Active Plan: `openspec/changes/add-long-short-rest/tasks.md`
- Linked Proposal: `openspec/changes/add-long-short-rest/proposal.md`
- Scope: Execute Phase 4 only. Do not proceed to subsequent phases.

## Implementation Steps
1. Update API types and client functions in `dungeoneer-frontend/lib/api/characters.ts`:
   - Add types for `RestType` ("SHORT" | "LONG"), `HitDiceSpendItem`, `RestPayload`, and response types.
   - Update `CharacterSheet` and `CharacterClassSummary` interfaces to include `exhaustion: number`, `hitDiceSpent: number`, and `availableHitDice: number`.
   - Add API client function `takeRest(characterId: string, payload: RestPayload): Promise<CharacterSheet>`.
2. Create `RestModal` component in `dungeoneer-frontend/components/character-sheet/rest-modal.tsx`:
   - Provide intuitive UI tabs or mode toggle for "Descanso Curto" (Short Rest) and "Descanso Longo" (Long Rest) in Portuguese (pt-BR).
   - Short Rest mode: display available Hit Dice per class, allow selecting number of dice to spend, display Constitution modifier bonus preview, and submit button.
   - Long Rest mode: display clear breakdown of benefits (full HP recovery, half Hit Dice restored, spell slots replenished, feat charges reset, exhaustion −1) with a confirmation button.
   - Invariant UX: disable rest actions when character is at 0 HP (unconscious/dead) with clear warning text; disable short rest dice submission when 0 dice are selected or available.
   - Integrate with TanStack Query / state mutation to update the character sheet on success.
3. Integrate rest controls and status badges into character sheet page `dungeoneer-frontend/app/characters/[id]/page.tsx`:
   - Add "Descanso Curto" and "Descanso Longo" action buttons in the character sheet header.
   - Display Hit Dice counter (e.g. `Dados de Vida: X/Y`) in the vitals/combat section.
   - Display exhaustion badge (e.g. `Exaustão: N`) prominently when `exhaustion > 0`.
4. Implement Vitest component tests in `dungeoneer-frontend/`:
   - Test `RestModal` rendering, mode switching, hit dice allocation stepper, and payload dispatch (TP-05).
   - Test disabled state behavior when character has 0 HP.

## Quality Gates & Verification
- Run local validation: `npm test` (inside `dungeoneer-frontend`)
- Commit locally: `git commit -m "feat(ui): add rest modal, hit dice controls, and exhaustion badge"`

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/add-long-short-rest/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 5.
