# Tasks: Level Up Half Feat Ability Choice

> **Change ID:** `level-up-half-feat-ability-choice`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Phase 1: API & DTO Alignment
- [x] 1.1 Inspect and update `LevelUpCharacterRequest` / DTOs in backend to support half-feat ability bonus selection (`featAbilityBonusChoice` or ability scores map).
- [x] 1.2 Update `LevelUpService` to apply the chosen half-feat ability bonus to the character's `AbilityScores` when a feat with `abilityBonusChoice` or `abilityBonuses` is selected.
- [x] 1.3 Add backend unit/integration tests verifying half-feat ability bonus application and validation.

## Phase 2: Frontend Level Up Wizard UI & State
- [x] 2.1 Update `LevelUpWizardModal` to detect if the selected feat is a half-feat (`abilityBonusChoice` or `abilityBonuses`).
- [x] 2.2 Render an interactive ability score selector when a half-feat is selected, restricted to `allowedAbilities`.
- [x] 2.3 Pass the selected ability bonus into the level-up mutation payload.
- [x] 2.4 Update frontend unit tests in `level-up-wizard.test.tsx` verifying half-feat ability selection rendering, validation, and payload submission.

## Phase 3: Verification & Integration
- [x] 3.1 Run backend tests (`mvn test -f player-character-service/backend/pom.xml`).
- [x] 3.2 Run frontend unit tests (`npm --prefix dungeoneer-frontend run test:unit`).
- [x] 3.3 Run frontend production build (`npm --prefix dungeoneer-frontend run build`).
- [x] 3.4 Rebuild Docker container `player-character-service` and verify live level-up with half-feat.

## Review Gate
- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
