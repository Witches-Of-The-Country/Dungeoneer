# Proposal: Level Up Half Feat Ability Choice

> **Change ID:** `level-up-half-feat-ability-choice`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-11-03
> **Last Updated:** 2026-11-03

---

## Summary

In D&D 5e / 2024, Half-Feats grant both special features and a +1 increase to an ability score chosen from a restricted set (e.g. `abilityBonusChoice` allowing 1 selection among `["STR", "DEX", "CON"]` or direct fixed `abilityBonuses`). Currently, in the Level-Up Wizard, selecting a Feat does not prompt the player to choose their ability score increase when a half-feat is selected, nor does the level-up submission payload convey this choice to update the character's persisted ability scores. This change introduces half-feat ability selection into the Level-Up Wizard UI, aligns the Level-Up request payload, and updates the backend domain service to apply the chosen bonus to the character's ability scores.

## Motivation

Players leveling up their characters to ASI levels (4, 8, 12, 16, 19) who choose a Half-Feat (such as *Athlete*, *Resilient*, *Fey Touched*, *Actor*, etc.) expect to gain both the feat's special capabilities and the designated +1 attribute boost. Without this selection UI and backend application, characters leveling up with half-feats miss their ability score increases, causing character sheet stat desynchronization and requiring manual DM/database interventions.

## Scope

### In Scope
- Inspect and support half-feat ability score choices in `LevelUpCharacterRequest` / DTOs.
- Update `LevelUpService` in `player-character-service` to validate and increment the selected ability score when a half-feat is chosen.
- Update `LevelUpWizardModal` in `dungeoneer-frontend` to inspect the selected feat's `mechanicalEffects.abilityBonusChoice` (or `abilityBonuses`), display an interactive selector for the granted attribute increase, require selection before completing level-up, and pass the choice to the backend.
- Unit and integration tests covering backend validation and frontend wizard interaction.

### Out of Scope
- Initial character creation wizard background origin feats (already handled or separate scope).
- Automated multiclass spell slot recalculation outside standard Level-Up.

## Proposed Solution

### Approach
1. **Backend Contract & Service**:
   - Verify whether `LevelUpCharacterRequest` has a field for ability score increases when a feat is selected (or whether `abilityScoreIncreases` can be populated alongside `selectedFeatId`).
   - In `LevelUpService`, if `selectedFeatId` corresponds to a feat with `abilityBonusChoice` or `abilityBonuses`, ensure the chosen ability is validated against `allowedAbilities` and added to the character's `AbilityScores`.
2. **Frontend UI**:
   - In `LevelUpWizardModal.tsx`, when the user picks the "Talento" option at an ASI level and selects a feat:
   - Check `selectedFeat.mechanicalEffects?.abilityBonusChoice` (or fixed `abilityBonuses`).
   - If present, render an accordion or card section "Bônus de Atributo do Talento (+1)" offering single-select radio chips for the allowed abilities (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
   - Block submission until the ability is selected if a choice is required.
   - Include the bonus in the payload sent to the backend.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java` | Modify | Validate half-feat `abilityBonusChoice` against `command.abilityScoreIncreases()`, enforce `allowedAbilities`, `chooseCount`, and `bonusAmount`, and apply increase to `AbilityScores`. |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/LevelUpServiceTest.java` | Modify | Unit tests verifying half-feat ability bonus application and 422 validation for non-allowed or excessive bonuses. |
| `dungeoneer-frontend/components/character-sheet/level-up-wizard-modal.tsx` | Modify | Inspect `selectedFeat.mechanicalEffects.abilityBonusChoice`, render interactive selection chips with current/projected score, enforce choice before advancing step 3, display in step 4 summary, and send in payload. |
| `dungeoneer-frontend/lib/api/encyclopedia.ts` | Modify | Ensure TypeScript typing for `AbilityBonusChoice` and `MechanicalEffects` supports half-feat properties. |
| `dungeoneer-frontend/__tests__/level-up-wizard.test.tsx` | Modify | Add unit tests for half-feat selection widget, step advancement gating, and mutation payload dispatch. |

### Data Model Changes (if applicable)
No new database migration required; `feats.mechanical_effects` already contains `abilityBonusChoice` and `abilityBonuses` from V26.

## Acceptance Criteria

- [ ] **Given** a character leveling up at an ASI level (e.g. Level 4) **When** the player chooses the "Talento" path and selects a half-feat with `abilityBonusChoice` (e.g. *Athlete* with STR or DEX) **Then** the UI displays an ability selection widget restricted to the allowed abilities.
- [ ] **Given** a half-feat selection without an ability chosen **When** the player attempts to advance or confirm Level Up **Then** the action is blocked with an informative validation message.
- [ ] **Given** a valid half-feat and ability selection **When** the Level-Up request is submitted **Then** the backend updates the character level, adds the feat to `character_feats`, and increases the chosen ability score in `ability_scores` by the designated bonus amount (typically +1).
- [ ] **Given** a feat without ability bonuses **When** selected **Then** no ability selection prompt is displayed and standard feat addition occurs.

## Defensive Security & Validation Considerations

### Input Validation
- Backend validates that the chosen ability belongs to `allowedAbilities` declared in the feat's `mechanicalEffects.abilityBonusChoice`.
- Rejects any unauthorized attribute increments using `CharacterValidationException` (RFC 7807 422 Unprocessable Entity).
- Character level and ownership (`X-User-Id`) continue to be enforced.

### Secret Isolation
- N/A — no secrets or external credentials involved.

### Authorization & Authentication Impact
- Existing `X-User-Id` header verification ensures only the character owner (or DM) can level up the character.

### Other Security Risks
- Prevents tampering where a client could attempt to increment arbitrary attributes or add +2 instead of +1 for half-feats.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Player selects half-feat in Level-Up Wizard and chooses allowed ability | Frontend Unit | Ability selector renders, radio selection works, payload includes bonus |
| TP-02 | Backend processes level-up with half-feat and ability bonus | Backend Unit/Integration | Character level increments, feat added, ability score increased |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Client sends half-feat ability bonus for non-allowed ability | Backend Unit | Throws `CharacterValidationException` 422 |
| TE-02 | Player leaves half-feat ability unselected in UI | Frontend Unit | Form validation prevents submission |

### Test Commands
- **Backend tests:** `mvn test -f player-character-service/backend/pom.xml`
- **Frontend tests:** `npm --prefix dungeoneer-frontend run test:unit`
- **Frontend build:** `npm --prefix dungeoneer-frontend run build`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Inconsistent payload shape between ASI and Half-Feats | Low | Med | Reuse or cleanly extend the existing DTO schema with explicit optional fields |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document half-feat ability score increase mechanics during level-up.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document Level-Up Wizard half-feat ability score selector UI behavior and validation.

## References
- `openspec/specs/player-character-domain.md`
- `openspec/specs/frontend-web-app.md`
