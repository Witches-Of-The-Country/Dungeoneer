# Bugfix Proposal: Background Origin Feats Restriction & Ability Score Reroll Rules

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Execution Order:** `004`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** Medium-High
> **Status:** draft
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Fixes two character creation and progression governance bugs:
1. Enforces that selecting a Background in the Character Creation Wizard strictly grants/limits selection to **Origin Feats (`featType == ORIGIN`)** in both the frontend UI, database background seeds, and backend `CreateCharacterService` validation.
2. Implements **Ability Score Reroll Governance**: allows free ability score rerolling on basic unbound character sheets (`campaignId == null`), while requiring explicit **Dungeon Master (Mestre) authorization (`allowAbilityReroll`)** when a character is bound to a campaign (`campaignId != null`).

## Grouped Bugs & Symptoms

- **Bug 1:** Ao escolher antecedentes, as feats de origem não estão se limitando às feats de origem, dá pra escolher qualquer uma.
- **Bug 2:** Quando personagem estiver vinculado a campanha, mestre decide se pode re-rolar atributos; na ficha de personagem básica permitir re-rolar.

## Root Cause Analysis (Causa Raiz)

1. **Unrestricted Feats on Background Selection**:
   - **Database Seed Mismatch (`V3` vs `V14`/`V18`)**: In `V3__seed_srd_2014_and_2024_reference_data.sql` (lines 93–98), `player_character.backgrounds.origin_feat_name` stores English names with parentheticals (e.g., `'Magic Initiate (Cleric)'`), whereas `player_character.feats` stores localized names (`'Iniciado em Magia'`, English `'Magic Initiate'`). In `app/characters/new/page.tsx` (lines 465–482), the name lookup fails (`matched == undefined`) and falls back to an open `<select>`.
   - **Legacy Feat Classification (`V14`/`V16`)**: Several feats in the database were seeded before `feat_type` was introduced or lacked strict `ORIGIN` vs `GENERAL` classification.
   - **Missing Backend Validation (`CreateCharacterService.java`)**: Lines 146–153 accept any `command.originFeatId()` without checking `f.getFeatType() == FeatType.ORIGIN`.
2. **Ability Score Reroll Lock**:
   - In `RollService.java` (lines 50–61), `abilityRollRepository.findUnusedByUserId(userId)` locks a user into a single cached 4d6kh3 roll.
   - In `app/characters/new/page.tsx` (line 672), `if (hasRolledAbilities) return;` blocks rerolling during creation.
   - In `app/characters/[id]/edit/page.tsx` (lines 151–199), ability scores are rendered as read-only ("Atributos Imutáveis de Criação"), with no backend endpoint or campaign permission flag (`allow_ability_reroll`) allowing unbound characters to reroll freely or campaign DMs to grant reroll permission.

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

- **Derived Stats Recalculation on Ability Reroll**: When a character rerolls ability scores on their sheet, changing `CON` alters `hitPointsMax` (and clamps `hitPointsCurrent`), changing `DEX` alters `armorClass` and `initiativeBonus`, and changing `CHA`/`WIS`/`INT` alters spell save DCs and `ABILITY_MODIFIER` feat max charges. `CharacterSheetHydrator` and the reroll use case must recalculate these derived stats atomically.
- **Level-Up ASI vs Creation Origin Feat**: Level-up ASI steps (`LevelUpWizardModal.tsx` at levels 4, 8, 12, 16, 19) rightfully allow `GENERAL` and `ORIGIN` feats, whereas Level 1 Background feat selection must strictly allow `ORIGIN` feats only.

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Table | Discrepancy Identified | Adaptation / Migration Action |
|-------------------------|------------------------|-------------------------------|
| `player_character.backgrounds` | `origin_feat_name` holds unmatched strings (`'Magic Initiate (Cleric)'`, etc.) | Flyway migration `V31__align_origin_feats_and_reroll_permissions.sql` updates `origin_feat_name` to match canonical `feats` names/slugs (`Iniciado em Magia`, `Alerta`, `Vigoroso`, `Habilidoso`, `Atacante Selvagem`). |
| `player_character.feats` | Non-origin feats potentially missing strict `feat_type` separation | Flyway `V31` audits and enforces `feat_type = 'ORIGIN'` strictly on SRD 2024 Origin Feats (`Alert`, `Crafter`, `Healer`, `Lucky`, `Magic Initiate`, `Musician`, `Savage Attacker`, `Skilled`, `Tavern Brawler`, `Tough`) and `'GENERAL'` on general feats. |
| `player_character.characters` | Lacks `allow_ability_reroll` column | Flyway `V31` adds `allow_ability_reroll BOOLEAN NOT NULL DEFAULT FALSE` to `player_character.characters`. |

## Scope

### In Scope
- Create Flyway migration `V31__align_origin_feats_and_reroll_permissions.sql` aligning background `origin_feat_name`, auditing `feat_type = 'ORIGIN'`, and adding `allow_ability_reroll` to `characters`.
- Add strict validation in `CreateCharacterService.java` verifying `f.getFeatType() == FeatType.ORIGIN` for `originFeatId`.
- Filter and bind Origin Feats strictly to `feat.featType === 'ORIGIN'` in `app/characters/new/page.tsx`.
- Update `RollService.java` and add `POST /api/characters/{id}/reroll-ability-scores` and `PATCH /api/characters/{id}/reroll-permission` (for Campaign DM):
  - If `character.getCampaignId() == null`: allow rerolling ability scores freely.
  - If `character.getCampaignId() != null`: allow rerolling only when `character.isAllowAbilityReroll() == true` (and automatically reset `allowAbilityReroll = false` after the reroll is consumed, or let the DM toggle it).
- Add UI controls on the Character Sheet / Edit page for rerolling attributes and DM permission toggle in the Campaign view.

### Out of Scope
- Point-buy custom house rules (e.g., 35-point buy).

## Proposed Solution

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V31__align_origin_feats_and_reroll_permissions.sql` | Create | Align background `origin_feat_name`, enforce `ORIGIN` feat types, add `allow_ability_reroll` column |
| `player-character-service/.../application/service/CreateCharacterService.java` | Modify | Enforce `f.getFeatType() == FeatType.ORIGIN` |
| `player-character-service/.../application/service/RollService.java` | Modify | Allow rerolling for unbound characters or DM-authorized campaign characters |
| `player-character-service/.../domain/model/PlayerCharacter.java` | Modify | Add `allowAbilityReroll` field and `updateAbilityScores` with HP recalculation |
| `player-character-service/.../adapter/in/web/controller/PlayerCharacterController.java` | Modify | Expose endpoints to reroll character ability scores and toggle DM reroll permission |
| `dungeoneer-frontend/app/characters/new/page.tsx` | Modify | Strictly filter Origin Feats (`featType === 'ORIGIN'`) and auto-select background's origin feat |
| `dungeoneer-frontend/app/characters/[id]/edit/page.tsx` | Modify | Enable ability score reroll when `campaignId == null` or `allowAbilityReroll == true` |

## Acceptance Criteria

- [ ] **Given** a player in the Character Creation Wizard selecting a Background **When** viewing the Origin Feat selector **Then** only feats with `featType == 'ORIGIN'` are selectable, and submitting a `GENERAL` feat ID to `POST /api/characters` is rejected with `400 Bad Request` / `422`.
- [ ] **Given** a basic character sheet NOT bound to any campaign (`campaignId == null`) **When** the player opens the attribute reroll option **Then** they can reroll and reassign their ability scores freely.
- [ ] **Given** a character bound to a campaign (`campaignId != null`) with `allowAbilityReroll == false` **When** the player attempts to reroll ability scores **Then** the button is locked with a message indicating the DM must authorize it, and the backend rejects unauthorized reroll requests.
- [ ] **Given** a character bound to a campaign where the DM enables `allowAbilityReroll = true` **When** the player rerolls ability scores **Then** the new scores and recalculated Max HP / AC / modifiers are saved.

## Defensive Security & Validation Considerations

### Input Validation
- Backend validates `f.getFeatType() == FeatType.ORIGIN` on character creation.
- Ability score values validated within D&D bounds (`1..30`, standard roll pool `3..18`).

### Secret Isolation
- N/A.

### Authorization & Authentication Impact
- Toggling `allowAbilityReroll` on a campaign-bound character verifies that the caller is the Campaign DM (or character owner when unbinding).
- Executing a reroll verifies that `X-User-Id` owns the character AND (`campaignId == null || allowAbilityReroll == true`).

### Other Security Risks
- Prevents players in a campaign from bypassing UI locks via direct API calls to inflate their ability scores.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Create character with a `GENERAL` feat passed as `originFeatId` | Unit (`CreateCharacterServiceTest`) | Throws `CharacterValidationException` |
| TR-02 | Reroll ability scores on unbound character (`campaignId == null`) | Unit / Integration | Ability scores and derived Max HP update successfully |
| TR-03 | Reroll ability scores on campaign-bound character (`allowAbilityReroll == true`) | Unit / Integration | Reroll succeeds and resets permission flag |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Player attempts `POST /reroll-ability-scores` when `campaignId != null` and `allowAbilityReroll == false` | Unit / Integration | Throws exception (`400`/`403`) |

### Test Commands
- **Backend tests:** `mvn clean test`
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| CON modifier reduction on reroll dropping Max HP below Current HP | Med | Med | Automatically recalculate `hitPointsMax` and clamp `hitPointsCurrent = Math.min(hitPointsCurrent, newHitPointsMax)` |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document strict `FeatType.ORIGIN` validation on creation and ability score reroll rules (`campaignId == null || allowAbilityReroll`).

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document `allow_ability_reroll` column and `V31` background origin feat alignment.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document Wizard Origin Feat filtering and Character Sheet / Campaign DM reroll UI controls.
