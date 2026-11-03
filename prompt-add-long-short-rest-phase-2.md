Focus: Backend / Domain Logic & Application Service
Task: Execute Phase 2: RulesEngine Rest Mechanics & RestService Implementation

## Execution Context
- Active Plan: `openspec/changes/add-long-short-rest/tasks.md`
- Linked Proposal: `openspec/changes/add-long-short-rest/proposal.md`
- Scope: Execute Phase 2 only. Do not proceed to subsequent phases.

## Implementation Steps
1. Extend `RulesEngine` interface and implementations (`Srd2014RulesEngine`, `Srd2024RulesEngine`) in `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/`:
   - Define rest strategy methods for short rest (Hit Dice healing calculation with CON modifier, minimum 0 per die, max HP cap) and long rest (full HP recovery, temporary HP preservation, recovering half total level Hit Dice rounded down with minimum 1, exhaustion decrement by 1 to minimum 0).
   - Maintain pure domain purity (zero framework dependencies).
2. Implement short-rest and long-rest recovery interactions in domain entities:
   - Enforce the non-resurrection invariant: characters at 0 HP (dead/unconscious) cannot rest; throw a domain exception or domain validation failure.
   - Integrate feat charge recovery via existing `FeatChargeCalculator` / `CharacterFeat.recoverCharges` based on rest recovery type.
   - For long rest, restore all expended spell slots.
   - Preserve temporary hit points on long rest (do not reset `temporaryHitPoints` to 0).
3. Rewrite `RestService` in `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/`:
   - Delegate rest mechanics resolution to the character's associated `RulesEngine` (`SRD_2014` or `SRD_2024`).
   - Validate character state before resting (check 0 HP invariant).
   - In Short Rest: validate that requested spends per `classProgressionId` do not exceed available Hit Dice, perform authoritative server-side die rolls, calculate healing via `RulesEngine`, apply healing to `currentHitPoints` (capped at `maxHitPoints`), update `hitDiceSpent` on class progressions, and recover short-rest feat charges.
   - In Long Rest: restore `currentHitPoints` to `maxHitPoints` while preserving `temporaryHitPoints`, recover Hit Dice across progressions up to the rule-calculated amount, reset all spell slots, recover long-rest feat charges, and decrement exhaustion.
4. Implement unit tests in `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/`:
   - Test RulesEngine rest methods for SRD 2014 and SRD 2024 (TE-07).
   - Test `RestServiceTest` covering short rest Hit Dice healing and spending (TP-01), healing capping at max HP (TE-01), long rest recovery of HP/HD/slots/charges/exhaustion (TP-02), half-HD rounding with min 1 (TE-05), exhaustion minimum 0 (TE-06), dead character rest rejection (TE-03), and temporary HP preservation.

## Quality Gates & Verification
- Run local validation: `mvn test -pl player-character-service/backend -Dtest=*Rest*Test,*RulesEngineTest`
- Commit locally: `git commit -m "feat(service): implement short and long rest mechanics with RulesEngine strategy"`

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/add-long-short-rest/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 3.
