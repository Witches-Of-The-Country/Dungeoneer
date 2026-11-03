Focus: Backend / Spring Boot 4 (Java 21) / Clean Architecture / TDD
Task: Execute Phase 1: Backend API & DTO Alignment, Validation, and Unit Tests

## Execution Context
- Active Plan: `openspec/changes/level-up-half-feat-ability-choice/tasks.md`
- Linked Proposal: `openspec/changes/level-up-half-feat-ability-choice/proposal.md`
- Domain Spec Delta: `openspec/changes/level-up-half-feat-ability-choice/specs/player-character-domain-delta.md`
- Scope: Execute Phase 1 only (Tasks 1.1, 1.2, 1.3). Align backend DTOs, implement validation and ability score application logic in `LevelUpService`, and add comprehensive unit tests. Do not proceed to frontend or subsequent phases.

## Implementation Steps
1. **DTO & Command Alignment:**
   - Inspect `LevelUpCharacterRequest` and `LevelUpCommand` in `player-character-service/backend` to verify support for ability score increases alongside feat selection.
   - Ensure `abilityScoreIncreases` (or equivalent map/list) is accepted and forwarded to the application use case when a feat is chosen.
2. **Domain Service Validation & Ability Score Application:**
   - In `LevelUpService`, inspect the selected feat's mechanical effects:
     - **Feats with `abilityBonusChoice`**: Validate that `command.abilityScoreIncreases()` contains exactly `chooseCount` (typically 1) attribute entry from `allowedAbilities`, each with bonus value equal to `bonusAmount` (typically +1). Reject missing selections, non-allowed attributes, or invalid bonus quantities by throwing `CharacterValidationException` (HTTP 422).
     - **Feats with fixed `abilityBonuses`**: Validate or apply declared fixed bonuses.
     - **Feats without ability bonuses**: Reject requests supplying `abilityScoreIncreases` alongside a non-half feat with `CharacterValidationException`.
   - Apply valid attribute bonuses to the character's `AbilityScores`, enforcing the standard attribute maximum cap of 20.
3. **Unit Tests & Quality Gates:**
   - Update `LevelUpServiceTest` to cover happy paths (half-feat with valid ability choice correctly increments character stats and saves feat) and failure modes (disallowed attribute choice, empty choice, excessive bonus amount, bonus provided for non-half feat, capping at 20).
   - Ensure all domain validation rules map to RFC 7807 422 status codes.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git add player-character-service/backend/
  git commit -m "feat(character): validate and apply half-feat ability bonus on level up"
  ```

## Phase Handoff & Cumulative Walkthrough
Update or create the shared walkthrough artifact (`openspec/changes/level-up-half-feat-ability-choice/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test execution results.
3. Handoff context prepared for Phase 2 (Frontend Level Up Wizard UI & State).
Pause and prompt the user to commit progress before proceeding to Phase 2.
