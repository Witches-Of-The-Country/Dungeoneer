# Bugfix Proposal: Fix Legacy Items, Armors, Feat Mechanics & Rest Charge Recovery

> **Change ID:** `002-fix-legacy-items-and-feats-mechanics`
> **Execution Order:** `002`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** High
> **Status:** draft
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Fixes five interconnected mechanical bugs across items, armors, class features/feats, combat actions, and short/long rests:
1. Repairing missing AC mechanics on Half Plate ("Armadura de Placa Parcial") and 5 other SRD armors omitted in `V18`.
2. Introducing `WondrousMechanics` and effective ability score / passive AC calculation for items that set ability scores or grant ability/AC bonuses.
3. Ensuring rolling Bardic Inspiration ("Inspiração Bárdica") and other rollable charged features spends a charge.
4. Normalizing legacy JSON keys (`maxUses`, `rechargeOn`) on Channel Divinity ("Canalizar Divindade") and 14+ other class features so their action buttons and charge counters render and work on the character sheet.
5. Fixing the Hibernate JPA cascade overwrite bug in `RestService.takeRest` that reverted recovered feat charges back to unrecovered state during Short and Long Rests.

## Grouped Bugs & Symptoms

- **Bug 1:** Armadura de placa parcial não atualiza CA — possivelmente coisa de itens legado não tendo mecânicas ativas.
- **Bug 2:** Itens que dão `set` no valor ou adicionam bônus de atributo não funcionando (talvez seja coisa de item legado).
- **Bug 3:** Rolar Inspiração Bárdica não está gastando a carga (quando usa no menu ações) -> investigar se é questão de feat legado.
- **Bug 4:** Canalizar Divindade sem ação (não tem botão de rolar, usar, nadinha).
- **Bug 5:** Descanso curto/longo não recuperando cargas.

## Root Cause Analysis (Causa Raiz)

1. **Half Plate & Missing SRD Armors (`V13` vs `V18`)**:
   In `V13__seed_complete_srd_items.sql`, Half Plate was seeded with slugs `'half-plate-armor-2014'` and `'half-plate-armor-2024'`. However, `V18__populate_srd_mechanical_properties.sql` (line 91) executed `WHERE slug IN ('half-plate-2014', 'half-plate-2024')`, matching `0` rows and leaving `mechanical_properties = '{}'`. Additionally, `padded-armor`, `hide-armor`, `scale-mail`, `ring-mail`, and `splint-armor` were completely omitted from `V18`. When equipped, `CharacterSheetHydrator` deserializes them as `EmptyItemMechanics` and ignores them in AC calculation.
2. **Wondrous Items Setting/Adding Ability Scores**:
   `ItemMechanics` is a sealed interface only allowing `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, and `EmptyItemMechanics`. Items like *Gauntlets of Ogre Power*, *Headband of Intellect*, *Amulet of Health*, *Belts of Giant Strength*, *Ioun Stones*, *Manuals/Tomes*, *Cloak/Ring of Protection*, and *Bracers of Defense* have `mechanical_properties = '{}'` in `V13`/`V24`, and `CharacterSheetHydrator` never calculates effective ability scores or passive item AC bonuses.
3. **Bardic Inspiration Roll Not Spending Charges**:
   In `V17`, Bardic Inspiration has `diceFormula: "1d6"`. In `actions-tab.tsx` (line 289), the "Gastar" button has the guard `!isRollable`, hiding it whenever a dice formula exists. Clicking "Rolar" opens `DiceRollerModal`, which only calls `useFeatMutation.mutate(activeRoll.actionId)` inside `handleApplyHealing` (`isHealing == true`). Non-healing rollable feats never trigger the charge deduction endpoint.
4. **Channel Divinity Missing Action Button**:
   In `V20__seed_complete_srd_classes_and_subclasses_progression.sql` (line 67), Channel Divinity was seeded with JSON `{"grantsAction":true,"actionType":"ACTION","actionName":"Canalizar Divindade","maxUses":1,"rechargeOn":"SHORT_REST"}`. However, `MechanicalEffects.java` expects `fixedMaxCharges` / `usesPerRest` and `restType`. Jackson ignores `maxUses` and `rechargeOn`, resulting in `maxCharges = null`. In `actions-tab.tsx`, an action with `maxCharges == null` and no dice formula renders neither "Gastar" nor "Rolar".
5. **Short/Long Rest JPA Cascade Overwrite (`RestService.java`)**:
   In `RestService.takeRest` (lines 104–163), `recoverFeats(...)` updates and saves each `CharacterFeat` in the repository, but does not update `characterToSave.getFeats()`. Immediately after `recoverFeats(...)`, `characterRepository.save(characterToSave)` is called. Because `PlayerCharacterJpaEntity.characterFeats` has `cascade = CascadeType.ALL`, Hibernate cascades the stale `characterToSave.getFeats()` list over the database, wiping out all recovered charges!

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

- **5 Additional Broken SRD Armors**: Padded Armor (`baseAc: 11, LIGHT`), Hide Armor (`baseAc: 12, MEDIUM`), Scale Mail (`baseAc: 14, MEDIUM`), Ring Mail (`baseAc: 14, HEAVY`), and Splint Armor (`baseAc: 17, HEAVY`) also had empty `mechanical_properties` and failed to update AC.
- **14+ Broken Class Features in `V20` & `V25`**: All features seeded with `maxUses` and `rechargeOn` failed charge calculation and recovery:
  - *Wild Shape* (`wild-shape-2014/2024`)
  - *Ki / Monk Focus* (`ki-monk-2014`, `monk-focus-2024`)
  - *Divine Intervention* (`divine-intervention-2014/2024`)
  - *Arcane Recovery* (`arcane-recovery-2014/2024`)
  - *Font of Magic / Sorcery Points* (`font-of-magic-2014`, `sorcery-points-2024`)
  - *Combat Superiority* (`combat-superiority-2014/2024`)
  - *Action Surge (2 uses)*, *Indomitable (2 & 3 uses)*, *Stroke of Luck*, *Eldritch Master*
  - *Preserve Life* (`channel-divinity-preserve-life-2014/2024`)

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Table | Discrepancy Identified | Adaptation / Migration Action |
|-------------------------|------------------------|-------------------------------|
| `player_character.items` | `half-plate-armor-*` and 5 other armors have `'{}'` mechanics; wondrous ability/AC items have `'{}'` | Flyway migration `V29__fix_legacy_items_and_feats_mechanics.sql` populates `ArmorMechanics` and `WondrousMechanics` JSONB on catalog rows. Existing character inventories reference `items(id)` by FK and are repaired automatically. |
| `player_character.feats` | `V20` & `V25` class features use `maxUses` and `rechargeOn` instead of `fixedMaxCharges`, `usesPerRest`, `restType`, `shortRestRecoveryType`, `longRestRecoveryType` | Flyway `V29` normalizes JSONB keys in `feats` AND `MechanicalEffects.java` adds `@JsonAlias` support for backward compatibility with any custom/homebrew JSON. |
| `player_character.character_feats` | Existing characters already in DB have `max_charges = NULL` and `current_charges = NULL` for `V20`/`V25` features | Flyway `V29` executes an `UPDATE player_character.character_feats` backfill joining `feats` and `characters` to populate `max_charges` and `current_charges` for all existing character sheets. |

## Scope

### In Scope
- Create `WondrousMechanics` record implementing `ItemMechanics` (`abilityBonuses`, `abilityOverrides`, `acBonus`, `savingThrowBonus`).
- Update `ItemJpaEntity` and `CharacterSheetHydrator` to compute effective ability scores (`Math.max(base + bonus, override)`) and passive AC/Save bonuses from equipped (and attuned, if required) items.
- Add `@JsonAlias` fallback parsing in `MechanicalEffects` and `FeatJpaEntity` for `maxUses` and `rechargeOn`.
- Fix `RestService.takeRest` so `recoverFeats` returns the updated `List<CharacterFeat>` and updates `characterToSave = characterToSave.withFeats(recoveredFeats)` before calling `characterRepository.save(characterToSave)`.
- Create Flyway migration `V29__fix_legacy_items_and_feats_mechanics.sql` to fix all 6 SRD armors, wondrous ability/AC items, `V20`/`V25` feat JSON keys, and backfill `character_feats`.
- Update `actions-tab.tsx` and `dice-roller-modal.tsx` so rollable actions with charges (like Bardic Inspiration) spend a charge on roll and also expose a direct "Gastar" button.

### Out of Scope
- Equipment slot restrictions (handled in `003-fix-equipment-slots-and-combat-sheet`).

## Proposed Solution

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V29__fix_legacy_items_and_feats_mechanics.sql` | Create | Fix 6 SRD armors, wondrous items, V20/V25 feat JSONB keys, and backfill `character_feats` |
| `player-character-service/.../domain/model/ItemMechanics.java` | Modify | Permit `WondrousMechanics` in sealed interface |
| `player-character-service/.../domain/model/WondrousMechanics.java` | Create | Value object for ability score overrides, bonuses, AC bonus, and save bonus |
| `player-character-service/.../domain/model/MechanicalEffects.java` | Modify | Support legacy JSON aliases (`maxUses`, `rechargeOn`) |
| `player-character-service/.../adapter/out/persistence/ItemJpaEntity.java` | Modify | Serialize/deserialize `WondrousMechanics` |
| `player-character-service/.../application/service/CharacterSheetHydrator.java` | Modify | Compute effective `AbilityScores` and passive `acBonus` from equipped/attuned items |
| `player-character-service/.../application/service/RestService.java` | Modify | Attach recovered feats to `characterToSave` via `.withFeats(recovered)` before saving |
| `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` | Modify | Show "Gastar" button alongside "Rolar" when `maxCharges > 0` and pass charge deduction handler to `DiceRollerModal` |
| `dungeoneer-frontend/components/character-sheet/dice-roller-modal.tsx` | Modify | Trigger `useFeatMutation` when rolling a charged feat action |

## Acceptance Criteria

- [ ] **Given** a character equipping "Armadura de Placa Parcial" (Half Plate) or any of the 5 previously omitted armors **When** the character sheet hydrates **Then** `armorClass` updates according to the armor's base AC and DEX cap (`15 + min(DEX, 2)` for Half Plate).
- [ ] **Given** a character equipping an item that sets an ability score (e.g., *Gauntlets of Ogre Power* setting STR to 19) or adds an ability/AC bonus **When** the item is equipped (and attuned if required) **Then** the effective ability score, modifier, skills, saving throws, and AC reflect the item's mechanics.
- [ ] **Given** a Bard with "Inspiração Bárdica" **When** they click "Rolar" or "Gastar" in the Actions tab **Then** 1 charge is deducted from `currentCharges`.
- [ ] **Given** an existing or new Cleric/Paladin with "Canalizar Divindade" (or Druid with "Forma Selvagem", Monk with "Ki", etc.) **When** viewing the Actions tab **Then** the action displays its charge counter (`1/1` or `2/2`) and an interactive "Gastar" button.
- [ ] **Given** a character with spent charges on Short Rest or Long Rest features **When** they execute a Short or Long Rest **Then** `RestService` persists the recovered charges without Hibernate cascade overwriting them.

## Defensive Security & Validation Considerations

### Input Validation
- Charge deduction validates `currentCharges > 0` in domain logic before decrementing.
- Ability score overrides/bonuses are computed server-side in `CharacterSheetHydrator` from trusted catalog/homebrew item mechanics.

### Secret Isolation
- N/A — no secrets or credentials involved.

### Authorization & Authentication Impact
- Character rest and feat usage endpoints continue enforcing character ownership via `X-User-Id`.

### Other Security Risks
- Prevents client-side manipulation of derived AC or ability modifiers by keeping calculation strictly in `CharacterSheetHydrator`.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Equip Half Plate (`half-plate-armor-2024`) on character with DEX 14 | Unit / Integration | Hydrated AC equals `17` (`15 + 2`) |
| TR-02 | Equip Gauntlets of Ogre Power (STR=19) on character with base STR 10 | Unit | Hydrated effective STR is `19` (`+4` modifier) |
| TR-03 | Take Short Rest / Long Rest after spending feat charges | Integration | `character_feats.current_charges` is restored and survives `characterRepository.save()` |
| TR-04 | Roll Bardic Inspiration in `ActionsTab` | Frontend Unit (Vitest) | `useFeatMutation` is called and charge decrements |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Character has base STR 20 and equips Gauntlets of Ogre Power (override 19) | Unit | Effective STR remains `20` (`Math.max(base, override)`) per D&D 5e rules |
| TE-02 | Legacy feat JSON with `{"maxUses": 2, "rechargeOn": "SHORT_REST"}` | Unit | Deserializes with `maxCharges = 2` and `restType = SHORT` |

### Test Commands
- **Backend tests:** `mvn clean test`
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Ability score override lowering a character's naturally higher score | Med | Med | Use `Math.max(baseWithBonuses, overrideScore)` as required by D&D 5e rules |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document `WondrousMechanics`, effective ability score computation, and `RestService` cascade-safe feat recovery.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document migration `V29` normalizing SRD armor mechanics, wondrous item mechanics, feat JSON keys, and `character_feats` backfill.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document charge deduction on rollable feat actions and "Gastar" button visibility in `ActionsTab`.
