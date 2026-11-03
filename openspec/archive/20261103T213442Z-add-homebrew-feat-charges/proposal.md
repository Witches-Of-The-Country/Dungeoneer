# Proposal: Add Homebrew Feat Charges

> **Change ID:** `add-homebrew-feat-charges`
> **Status:** draft
> **Author:** <!-- @handle -->
> **Created:** 2026-11-03
> **Last Updated:** 2026-11-03

---

## Summary

Adds a configurable charge system to homebrew feats. Max charges are derived from a formula (`FIXED`, `ABILITY_MODIFIER`, `PROFICIENCY`, `LEVEL`) and recovery is configured independently for short and long rest. This change is a prerequisite of `add-long-short-rest`, which consumes the recovery rules.

## Motivation

Today `MechanicalEffects`/`CharacterFeat` only support a static `usesPerRest` + `restType`, with recovery hardcoded to long rest. Homebrew authors need D&D-style "modifier/proficiency/level times per day" and flexible per-rest recovery.

## Scope

### In Scope
- Charge max formula: `FIXED` (N), `ABILITY_MODIFIER` (author-chosen ability STR/DEX/CON/INT/WIS/CHA, minimum 1), `PROFICIENCY` (proficiency bonus), `LEVEL` (character level).
- Recovery rule per rest type (short and long, independent): `NONE`, `ALL`, `FIXED` (X), `MODIFIER`, `PROFICIENCY`, `LEVEL`. Long rest default = `ALL`; short rest default = `NONE`.
- Dynamic recalculation of max charges (level up, ability change); `currentCharges` clamped to max; recovery never exceeds max.
- Backward compatibility with existing feats (`usesPerRest`/`restType`) via JSON defaults/mapping.
- Homebrew feat creation/edit UI and API; charge display on feat cards.

### Out of Scope
- Rest endpoints/buttons and applying recovery (`add-long-short-rest`).
- Charges for spells, items or class resources.
- Changing SRD seed feats (only mapped by compatibility defaults).

## Proposed Solution

### Approach
Extend the `MechanicalEffects` record (JSONB `feats.mechanical_effects`) with structured fields: `maxChargesType`, `chargeAbility`, `fixedMaxCharges`, `shortRestRecoveryType`, `shortRestRecoveryFixed`, `longRestRecoveryType`, `longRestRecoveryFixed`. Domain stays framework-free: a pure calculator resolves max charges and recovery amounts from character stats. `CharacterSheetHydrator` recomputes `maxCharges` on each hydration and clamps `currentCharges`; `CharacterFeat.recoverCharges` takes the rest type and stats. Legacy fields map to `FIXED` max + `ALL` recovery on the matching rest type.

Validation lives in `CreateHomebrewFeatRequest` (Jakarta Validation) plus cross-field checks in `HomebrewService` (e.g. `chargeAbility` required for `ABILITY_MODIFIER`, `fixed*` > 0 when type is `FIXED`).

### Impacted Files
<!-- Populated by sdd-code-explorer (report received). Backend root: player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter -->
| File | Change Type | Notes |
|------|-------------|-------|
| `domain/model/MechanicalEffects.java` | Modify | Charge and recovery config fields; new enums `ChargeType`, `ChargeRecoveryType` |
| `domain/model/CharacterFeat.java` | Modify | `recoverCharges` with independent short/long rules and clamping |
| `application/service/CharacterSheetHydrator.java` | Modify | Dynamic max recalculation + clamp |
| `application/service/CreateCharacterService.java` | Modify | Initialize charges from stats |
| `application/service/LevelUpService.java` | Modify | Initialize charges for newly granted feats |
| `application/service/HomebrewService.java` | Modify | Persist and validate config |
| `application/dto/CreateHomebrewFeatRequest.java` | Modify | Validated charge config |
| `application/dto/FeatDto.java` | Modify | Expose config |
| `application/dto/CharacterFeatDto.java` | Modify | Expose dynamic max and recovery metadata |
| `adapter/out/persistence/FeatJpaEntity.java` | Modify | JSONB compat for new fields |
| `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` | Modify | Charge builder form |
| `dungeoneer-frontend/lib/api/homebrew.ts` | Modify | Payload types |
| `dungeoneer-frontend/components/character-sheet/feats-tab.tsx` | Modify | Charges/recovery display |

### Data Model Changes
No new column: `feats.mechanical_effects` JSONB gains optional fields (missing fields default gracefully). A data migration is only needed if legacy mapping is persisted rather than resolved at read time (decision: resolve at read time, no migration).

## Acceptance Criteria

- [ ] **Given** a feat with `LEVEL` max charges, **When** the character levels up, **Then** `maxCharges` equals the new level on next hydration.
- [ ] **Given** `ABILITY_MODIFIER` with a modifier ≤ 0, **When** max is computed, **Then** max charges is 1.
- [ ] **Given** short recovery `FIXED 1` and 0/3 charges, **When** recovery is evaluated for a short rest, **Then** current becomes 1.
- [ ] **Given** recovery that would exceed max, **When** applied, **Then** current is clamped to max.
- [ ] **Given** long recovery unspecified, **Then** it defaults to `ALL`.
- [ ] **Given** `ABILITY_MODIFIER` without `chargeAbility`, **When** the homebrew feat is submitted, **Then** the API returns 400 ProblemDetail.
- [ ] **Given** a legacy feat with `usesPerRest`/`restType`, **When** hydrated, **Then** behavior is unchanged.

## Defensive Security & Validation Considerations

### Input Validation
- Jakarta Validation on `CreateHomebrewFeatRequest`; enums for types/abilities; `fixed*` bounded (1..99); cross-field validation in service.

### Secret Isolation
- No secrets involved.

### Authorization & Authentication Impact
- N/A — MVP has no authentication; no new endpoints.

### Other Security Risks
- JSONB deserialization limited to typed record fields; unknown enum values rejected (400).

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Max charges per formula (FIXED/MOD/PROF/LEVEL) | Unit | Correct values |
| TP-02 | Recovery per type, short and long | Unit | Correct amounts |
| TP-03 | Create homebrew feat with config | Integration/MockMvc | 201 + persisted JSONB |
| TP-04 | Dialog builds payload | Vitest | Expected payload |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Modifier ≤ 0 | Unit | Max = 1 |
| TE-02 | Recovery exceeds max | Unit | Clamped |
| TE-03 | Missing ability / invalid fixed | MockMvc | 400 ProblemDetail |
| TE-04 | Legacy JSON without new fields | Unit/Integration | Defaults applied |
| TE-05 | Max shrinks below current | Unit | Current clamped |

### Test Commands
- **Run tests:** `mvn clean test` (backend); `npm test` (frontend)
- **Coverage check:** `mvn clean verify` (JaCoCo ≥ 80%)

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Legacy feat behavior regression | Med | High | Read-time mapping + TE-04 |
| Overlap with `002` MechanicalEffects changes | Low | Med | Optional fields only |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Charge formulas, recovery rules, clamping.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** `mechanical_effects` JSONB structure.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Charge builder in homebrew dialog; feat card display.

## References

- Related change: `add-long-short-rest` (depends on this one)
- Prior changes: `002-advanced-homebrew-mechanics-and-prerequisites`, `mechanical-properties-and-class-progression`
