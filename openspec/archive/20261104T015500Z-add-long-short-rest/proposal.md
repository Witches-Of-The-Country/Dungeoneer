# Proposal: Add Long Short Rest

> **Change ID:** `add-long-short-rest`
> **Status:** draft
> **Author:** <!-- @handle -->
> **Created:** 2026-11-03
> **Last Updated:** 2026-11-03
> **Depends on:** `add-homebrew-feat-charges`

---

## Summary

Adds "Descanso Curto" and "Descanso Longo" to the character sheet, implementing D&D rest mechanics for `SRD_2014` and `SRD_2024` through `RulesEngine`: Hit Dice spending and healing, feat charge recovery, spell slots, exhaustion and Hit Dice recovery.

## Motivation

A prototype `RestService` exists but is incomplete: no Hit Dice, no exhaustion, no short-rest recovery, no spell slots, no `RulesEngine` delegation, and long rest wrongly zeroes temp HP.

## Scope

### In Scope
- **Short rest:** player chooses Hit Dice to spend per class; each heals die roll + CON mod (min 0), capped at max HP; recovers feat charges per their short-rest rule and class resources flagged for short rest.
- **Long rest:** full HP; temp HP preserved; recover half of total Hit Dice (min 1); all spell slots; feat charges per long rule; exhaustion −1 (min 0).
- Hit Dice and exhaustion persistence (`hit_dice_spent`, `exhaustion`).
- `RulesEngine` strategy methods implemented by `Srd2014RulesEngine` and `Srd2024RulesEngine`.
- Invariant: no resurrection — characters at 0 HP/dead cannot rest (error 409/422 ProblemDetail).
- Frontend: rest buttons, `RestModal`, Hit Dice counter, exhaustion badge.

### Out of Scope
- Charge formula model (`add-homebrew-feat-charges`).
- Exhaustion penalties/effects, death saves, resurrection spells.
- Interrupted rest rules, once-per-24h long rest limit, campaign time.

## Proposed Solution

### Approach
Rewrite `RestService.takeRest` to delegate to the character's `RulesEngine`. `RestCommand` carries `restType` and, for short rest, a list of `{classProgressionId, count}` Hit Dice spends; the die roll is performed server-side (validated, authoritative) and returned in the result. Domain gets `ClassProgression.spendHitDice/recoverHitDice` (0 ≤ spent ≤ class level) and `PlayerCharacter.exhaustion` (0–6). Flyway `V27` adds both columns. Spell slot recovery and feat recovery reuse existing domain/ `CharacterFeat.recoverCharges`.

Open points to confirm at review: server-side rolling vs client-submitted rolls (recommended server-side), whether class resources beyond feats exist for recovery (explorer found only feat charges), SRD 2024 differences (exhaustion −1 both; Warlock Pact slots on short rest if modeled).

### Impacted Files
<!-- Populated by sdd-code-explorer. Backend root: player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter -->
| File | Change Type | Notes |
|------|-------------|-------|
| `src/main/resources/db/migration/V27__add_exhaustion_and_hit_dice_tracking.sql` | Create | `characters.exhaustion`, `class_progressions.hit_dice_spent` |
| `domain/model/PlayerCharacter.java` | Modify | `exhaustion`, rest guards |
| `domain/model/ClassProgression.java` | Modify | `hitDiceSpent`, spend/recover |
| `domain/model/RulesEngine.java` | Modify | Rest strategy methods |
| `domain/model/Srd2014RulesEngine.java` | Modify | 2014 rest rules |
| `domain/model/Srd2024RulesEngine.java` | Modify | 2024 rest rules |
| `application/port/in/RestUseCase.java` | Modify | Contract |
| `application/dto/RestCommand.java` | Modify | Hit Dice spends (validated) |
| `application/service/RestService.java` | Modify | Full short/long logic; temp HP fix |
| `application/dto/CharacterSheetResult.java` | Modify | exhaustion, hit dice |
| `application/dto/CharacterClassSummaryDto.java` | Modify | hitDice, spent, available |
| `adapter/out/persistence/PlayerCharacterJpaEntity.java` | Modify | exhaustion |
| `adapter/out/persistence/ClassProgressionJpaEntity.java` | Modify | hit_dice_spent |
| `adapter/in/web/controller/PlayerCharacterController.java` | Modify | `POST /{id}/rest` |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | RestPayload, types |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Modify | Buttons, Hit Dice, exhaustion |
| `dungeoneer-frontend/components/character-sheet/rest-modal.tsx` | Create | Rest modal |

### Data Model Changes
`V27`: `characters.exhaustion INT NOT NULL DEFAULT 0` (CHECK 0–6); `class_progressions.hit_dice_spent INT NOT NULL DEFAULT 0` (CHECK ≥ 0).

## Acceptance Criteria

- [ ] **Given** HP 10/30, CON +2, 2 Hit Dice available, **When** short rest spends 1 d8, **Then** HP increases by roll+2 (capped at 30) and spent +1.
- [ ] **Given** spending more Hit Dice than available, **When** short rest requested, **Then** 400 ProblemDetail and no state change.
- [ ] **Given** temp HP 5, **When** long rest, **Then** HP = max and temp HP remains 5.
- [ ] **Given** total level 7 with 7 spent, **When** long rest, **Then** 3 recovered (half rounded down, min 1).
- [ ] **Given** level 1 with 1 spent, **Then** long rest recovers 1.
- [ ] **Given** exhaustion 2, **When** long rest, **Then** 1; at 0 stays 0.
- [ ] **Given** spent spell slots, **When** long rest, **Then** all restored; short rest leaves them.
- [ ] **Given** feats with short/long rules, **When** each rest, **Then** charges recover per rule, clamped.
- [ ] **Given** character at 0 HP/dead, **When** any rest, **Then** rejected, no changes.
- [ ] **Given** each of `SRD_2014`/`SRD_2024`, **When** rest, **Then** the matching `RulesEngine` is used.

## Defensive Security & Validation Considerations

### Input Validation
- `RestCommand` validated (Jakarta): `restType` required; spends ≥ 1, class progression must belong to the character, count ≤ available. Rolls generated server-side.

### Secret Isolation
- No secrets.

### Authorization & Authentication Impact
- N/A — MVP has no authentication; existing endpoint only. Ownership of `classProgressionId` verified against the character.

### Other Security Risks
- Prevent client-forged rolls/healing (server-side RNG); concurrent rests guarded by transaction / optimistic locking if present.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Short rest spend Hit Dice | Unit | Heal + spent updated |
| TP-02 | Long rest full effects | Unit | HP, HD, slots, exhaustion, charges |
| TP-03 | POST rest endpoints | MockMvc | 200 + updated sheet |
| TP-04 | Persistence of new columns | Integration (Testcontainers) | Values persisted |
| TP-05 | Rest modal / buttons | Vitest | Correct payload |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Heal exceeds max HP | Unit | Capped |
| TE-02 | Overspend Hit Dice | Unit/MockMvc | 400 |
| TE-03 | Dead / 0 HP rest | Unit/MockMvc | Rejected |
| TE-04 | Foreign classProgressionId | MockMvc | 400/404 |
| TE-05 | Half-HD rounding, min 1 | Unit | Correct |
| TE-06 | Exhaustion at 0 | Unit | Stays 0 |
| TE-07 | Ruleset 2014 vs 2024 | Unit | Correct engine |

### Test Commands
- **Run tests:** `mvn clean test` (backend); `npm test` (frontend)
- **Coverage check:** `mvn clean verify`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Depends on charge model not yet merged | Med | High | Implement after `add-homebrew-feat-charges` |
| Flyway number collision with V27 | Low | Med | Verify latest migration before merge |
| Behavior change: temp HP kept on long rest | Low | Low | Documented, tested |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Rest rules, RulesEngine strategy, invariants.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** V27 columns.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Rest buttons, RestModal, Hit Dice counter, exhaustion badge.

## References

- Depends on: `add-homebrew-feat-charges`
- Prior change: `mechanical-properties-and-class-progression`
