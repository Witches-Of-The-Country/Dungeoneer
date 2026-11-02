# Proposal: Mechanical Properties, Class Progression & Character Sheet Evolution

> **Change ID:** `mechanical-properties-and-class-progression`
> **Status:** approved
> **Author:** @jaca / Antigravity
> **Created:** 2026-10-31
> **Last Updated:** 2026-11-01

---

## Summary

This change introduces structured mechanical properties for items (weapons, armor/robes/shields, and consumables), a unified mechanical feature architecture for feats and class/subclass/lineage abilities across levels 1–20, complete SRD 2014 & 2024 database migrations and seeds (including all 13 SRD subclasses and class progression tables), an XP Tracker and unrestricted Multiclass Level Up Engine (UC2), and full interactive activation of the Character Sheet Dashboard tabs (`Inventário`, `Talentos`, and `Ações`), Homebrew creation forms, and Class Encyclopedia progression views.

## Motivation

Currently, items and feats in Dungeoneer store only descriptive text (`description`), lacking structured mechanical attributes necessary for automated character sheet calculations:
- **Items lack mechanical value**: Weapons do not have structured damage dice, damage types, versatile dice, or weapon properties; Armor lacks base AC, DEX modifier caps, and category classifications (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`); Consumables cannot be used directly from the sheet to roll dice, heal HP/grant Temp HP, and auto-decrement inventory quantity.
- **Classes & Subclasses lack progression features**: `Archetype` only stores base level-1 proficiencies and hit dice, while `Subclass.java` is an empty stub. Level 1–20 class features and subclass features are missing from both the SRD compendium and the Homebrew creator.
- **Level Up & XP progression (UC2) are unimplemented**: Players cannot track XP, trigger level-ups (via XP thresholds or Milestone), multiclass freely, roll or select HP increases per level, unlock level-gated class/subclass features, or choose between Ability Score Improvements (ASI) and Feats at ASI thresholds.
- **Character Sheet tabs are placeholders**: The `Inventário`, `Talentos`, and `Ações` tabs on `/characters/[id]` currently display "Em breve" placeholders instead of interactive inventory management, dynamic AC recalculation, feat charge tracking, and weapon/feature attack dice rollers.

## Scope

### In Scope
- **Mechanical Item Persistence & Polymorphic Domain Model**:
  - Preserve common columns in `player_character.items` and add `mechanical_properties JSONB` mapped to pure Java 21 domain Value Objects (`ItemMechanics` sealed interface implemented by `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`) with zero framework imports in `domain/model/`.
  - **Weapons (`WeaponMechanics`)**: Support composite damage rolls (`damageRolls`, e.g., `1d8 PIERCING` + `1d6 FIRE`), optional `versatileDice` (`1d10`), `magicBonus` (`+0` to `+3`), and weapon properties (`FINESSE`, `LIGHT`, `HEAVY`, `REACH`, `TWO_HANDED`, `THROWN`, `AMMUNITION`, `LOADING`, `VERSATILE`).
  - **Armor (`ArmorMechanics`)**: Support `baseAc`, `magicBonus`, `strengthRequirement`, `stealthDisadvantage`, and `armorCategory` (`LIGHT`: `baseAc + dexMod`; `MEDIUM`: `baseAc + min(dexMod, 2)`; `HEAVY`: `baseAc`; `ROBE`: `baseAc + dexMod` counting as unarmored for class rules and allowing additive AC bonuses; `SHIELD`: additive `+baseAc + magicBonus`).
  - **Consumables (`ConsumableMechanics`)**: Support `effectType` (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`), optional `diceFormula` (e.g., `2d4+2`), and `effectDescription`. Using a consumable executes a server-side roll, applies HP/Temp HP to the character, and decrements `quantity` by 1 (deleting the `character_items` row when `quantity == 0`).
- **Unified Feats & Class/Subclass/Lineage Feature Architecture**:
  - Add `feat_type VARCHAR(30)` (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`) and `mechanical_effects JSONB` to `player_character.feats`, mapped to the `MechanicalEffects` domain Value Object (ability score bonuses, passive bonuses `acBonus`, `initiativeBonus`, `speedBonus`, `hpPerLevelBonus`, granted proficiencies, active actions/dice formulas, and resource charges `usesPerRest` + `restType: SHORT | LONG`).
  - Create associative progression tables `player_character.archetype_feats (archetype_id, feat_id, unlock_level)` and `player_character.subclass_feats (subclass_id, feat_id, unlock_level)` covering levels 1–20.
  - Evolve `Subclass.java` from an empty stub into a full domain entity linked to `Archetype.java` (`subclassLevel` added to `archetypes`).
- **Complete SRD (2014 & 2024) Data Migration, Seed & Homebrew Support**:
  - Flyway migrations (`V16`–`V18`) altering schemas, extracting/populating `mechanical_properties` on existing SRD items and `mechanical_effects` on existing SRD feats, seeding all 13 SRD Subclasses across `SRD_2014` and `SRD_2024`, and seeding level 1–20 Class & Subclass Feats.
  - Expand `CreateHomebrewDialog` and backend `HomebrewService` to support dynamic mechanical forms for Weapons, Armor/Robes/Shields, Consumables, Feats with mechanical effects, and a hybrid Level 1–20 feature builder for Classes (`Archetype`) and Subclasses (supporting both creating new level-gated features inline with optional mechanical effects and linking existing catalog features).
  - Update `/encyclopedia/classes` so clicking any class card opens a rich Class Detail Modal displaying proficiencies, saving throws, the level 1–20 class progression table (with expandable feature descriptions and mechanical badges), and associated Subclasses.
- **Character Creation Above Level 1 & Career HP Method Lock**:
  - In the Character Creation Wizard (`/characters/new`), players choose only their **primary class** (removing multiclass selection from creation), always select their career `hpCalculationMethod` (`AVERAGE`, `ROLLED`, or `MAX`, governing levels 2–20 while Level 1 always receives maximum Hit Die + CON modifier), conditionally select a Subclass in creation **only** when `archetype.subclassLevel == 1` (e.g., `SRD_2014` Cleric, Sorcerer, and Warlock), and in `SRD_2024` select 1 **Origin Feat (`ORIGIN`)** during the Background step.
  - When a character is created with a target starting level $N > 1$, `CreateCharacterService` initializes the character at **Level 1** in the primary class (unlocking Level 1 `archetype_feats`, Level 1 `subclass_feats` if `subclassLevel == 1`, and the `SRD_2024` Origin Feat with its `MechanicalEffects`), and pre-populates `experiencePoints` with the official D&D 5e XP floor for level $N$ (e.g., Level 5 = `6,500 XP`), setting `hasPendingLevelChoices = true`.
  - Upon entering `/characters/[id]`, when `experiencePoints >= nextLevelXp` (or `hasPendingLevelChoices`), the sheet automatically opens `LevelUpWizardModal` sequentially (`Level 1 -> 2 -> ... -> N`), while also allowing the player to close and resume via the clickable `"Escolhas de Nível Pendentes"` badge or `"Subir de Nível!"` CTA.
- **XP Tracker, Level Up & Unrestricted Multiclassing Engine (UC2)**:
  - Add `experience_points INT NOT NULL DEFAULT 0` and `campaign_id UUID NULLABLE` to `player_character.characters`.
  - Add official D&D 5e XP progress bar (0 to 355,000 XP, levels 1–20) to the character sheet header with a `+ Adicionar XP` modal (disabled when `campaign_id != null`) and a pulsing `Subir de Nível!` CTA when XP reaches the next threshold, plus Milestone (`Marco`) level-up support that sets `experience_points` to the minimum XP floor of the new total level.
  - Multi-step **Level Up Wizard** supporting: advancing an existing class or adding any new class (**unrestricted multiclassing without ability score >= 13 prerequisite checks**), choosing a Subclass when reaching `subclass_level`, increasing Max HP strictly locked to the character's persisted `hpCalculationMethod` (server-computed `AVERAGE` or `MAX`, or server-validated single Hit Die roll token `hpRollId` from `hp_rolls` for `ROLLED`) + CON modifier + `hpPerLevelBonus` (e.g., *Tough*), unlocking Class/Subclass Feats for the new class level, and choosing between **ASI (+2 to one ability or +1/+1 to two abilities, capped at 20)** OR **selecting a `GENERAL` or `ORIGIN` Feat** (with informational-only prerequisites and automatic application of `abilityBonuses` and retroactive `hpPerLevelBonus`) at class ASI thresholds (`4, 8, 12, 16, 19`, plus Fighter `6, 14` and Rogue `10`).
- **Toggleable `DUNGEON_MASTER` Active Role in Profile & Navbar (`auth-service` & Frontend)**:
  - Keep `PLAYER` fixed as the mandatory base role while making `DUNGEON_MASTER` toggleable on/off in both `/profile` ("Papéis Ativos") and the `Navbar` profile dropdown via `POST /api/auth/me/roles/toggle-dm` in `auth-service`, persisting the updated role set and returning a freshly signed JWT token.
- **Full Activation of Character Sheet Dashboard Tabs (`/characters/[id]`)**:
  - **Inventário Tab**: Add/remove catalog items, adjust quantities, track total weight, toggle equip/unequip with automatic AC recalculation, manage attunement (`sintonização`, max 3 items), and trigger consumable use.
  - **Talentos Tab**: Display features grouped by Origin/General, Class, Subclass, and Lineage, with interactive charge expenditure and Short/Long Rest (`Descanso Curto` / `Descanso Longo`) recovery.
  - **Ações Tab**: Render auto-generated weapon attacks for equipped weapons (calculating attack roll bonus and damage rolls including versatile two-handed option) and active class/feat abilities with an interactive dice roller modal.

### Out of Scope
- Campaign management microservice (`campaign-service`) implementation — `campaign_id` on `characters` is introduced strictly as a nullable UUID placeholder for future campaign lock integration.
- Automated condition duration tracking across combat rounds (e.g., real-time initiative tracker turn timers).
- Enforcing ability score >= 13 prerequisites on multiclassing (explicitly excluded by design so players have unrestricted multiclassing freedom).

## Proposed Solution

### Approach
1. **Clean Architecture Polymorphic Domain Value Objects & JSONB Persistence**:
   In `player-character-service`, we preserve relational integrity for core columns while introducing PostgreSQL `JSONB` columns (`items.mechanical_properties` and `feats.mechanical_effects`) mapped in the persistence adapter via Hibernate `@JdbcTypeCode(SqlTypes.JSON)`. In `domain/model/`, pure Java 21 immutable Value Objects (`ItemMechanics` sealed interface with `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics` records/classes, and `MechanicalEffects`) enforce domain invariants (valid dice notation regex `^\d+d(4|6|8|10|12|20|100)([+-]\d+)?$`, valid `ArmorCategory`, non-negative bonuses) with zero framework imports.
2. **RulesEngine Strategy Evolution**:
   `RulesEngine` (`Srd2014RulesEngine` and `Srd2024RulesEngine`) is extended to compute dynamic Armor Class from equipped `ArmorMechanics` (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), character DEX/CON/WIS modifiers (preserving Unarmored Defense when unarmored or wearing a `ROBE`), and passive `acBonus` modifiers from attuned/equipped items and unlocked feats. Weapon attack bonuses and damage formulas are computed dynamically during sheet hydration (`CharacterSheetHydrator`) using STR or DEX (when `FINESSE` or ranged) plus proficiency bonus and `magicBonus`.
3. **Unified Progression & Level Up Use Case (UC2)**:
   `LevelUpService` orchestrates atomic level progression (up to total character level 20). It validates either sufficient `experiencePoints` for the next D&D 5e threshold or `milestone = true` (which sets `experiencePoints` to the new level's XP floor), increments an existing `ClassProgression` or appends a new multiclass `ClassProgression` (without ability score >= 13 restrictions), binds a `Subclass` when `classLevel == archetype.subclassLevel`, resolves HP gain (consuming a server-rolled `HpRoll` token or applying fixed average/max + CON mod + `hpPerLevelBonus`), auto-assigns unlocked `archetype_feats` and `subclass_feats` into `character_feats`, and applies ASI (+2 or +1/+1 up to 20) or a chosen Feat when the class level matches an ASI threshold.
4. **Interactive Character Sheet & Homebrew UI**:
   In `dungeoneer-frontend`, the "Em breve" placeholders in `/characters/[id]` are replaced by dedicated modular components (`InventoryTab`, `FeatsTab`, `ActionsTab`, `XpProgressBar`, `AddXpModal`, `LevelUpWizardModal`, `DiceRollerModal`) backed by TanStack Query mutations with optimistic/immediate cache invalidation. `CreateHomebrewDialog` dynamically renders mechanical sub-forms based on item/feat/class/subclass category, and `/encyclopedia/classes` displays the full 1–20 progression table and subclasses.

### Impacted Files

| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V16__add_mechanical_properties_and_class_progression_tables.sql` | Create | DDL for `items.mechanical_properties`, `feats.feat_type`, `feats.mechanical_effects`, `characters.experience_points`, `characters.campaign_id`, `archetypes.subclass_level`, and tables `subclasses`, `archetype_feats`, `subclass_feats`, `character_items` updates, `character_feats` updates |
| `player-character-service/backend/src/main/resources/db/migration/V17__seed_srd_subclasses_and_progression_feats.sql` | Create | Seed all 13 SRD subclasses (`SRD_2014` & `SRD_2024`) and level 1–20 class/subclass feats |
| `player-character-service/backend/src/main/resources/db/migration/V18__populate_srd_mechanical_properties.sql` | Create | Populate `mechanical_properties` on SRD weapons, armors, shields, potions and `mechanical_effects` on SRD feats |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ItemMechanics.java` | Create | Pure domain sealed interface for polymorphic item mechanics |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/WeaponMechanics.java` | Create | Domain VO for weapon damage rolls, versatile die, magic bonus, and weapon properties |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ArmorMechanics.java` | Create | Domain VO for armor category (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), base AC, magic bonus, STR req, stealth disadvantage |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ConsumableMechanics.java` | Create | Domain VO for consumable effect type (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`) and dice formula |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/DamageRoll.java` | Create | Domain VO pairing dice formula with `DamageType` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/DamageType.java` | Create | Domain enum for D&D 5e damage types |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/WeaponProperty.java` | Create | Domain enum for weapon properties (`FINESSE`, `VERSATILE`, `HEAVY`, etc.) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ArmorCategory.java` | Create | Domain enum (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ConsumableEffectType.java` | Create | Domain enum (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/FeatType.java` | Create | Domain enum (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RestType.java` | Create | Domain enum (`SHORT`, `LONG`, `NONE`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffects.java` | Create | Domain VO for feat passive bonuses, ASI grants, proficiencies, active actions, and rest charges |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterItem.java` | Create | Domain model for character inventory entry (`itemId`, `quantity`, `equipped`, `attuned`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterFeat.java` | Create | Domain model for unlocked character feat (`featId`, `unlockedAtLevel`, `currentCharges`, `maxCharges`, `choicesJson`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Item.java` | Modify | Add `ItemMechanics mechanicalProperties` field and invariants |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Feat.java` | Modify | Add `FeatType featType` and `MechanicalEffects mechanicalEffects` fields |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Subclass.java` | Modify | Implement full `Subclass` domain entity replacing empty 5-line stub |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Archetype.java` | Modify | Add `subclassLevel`, ASI levels helper, and progression feature bindings |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ClassProgression.java` | Modify | Support subclass binding and level advancement |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java` | Modify | Add `experiencePoints`, `campaignId`, `items`, `feats`, `addExperience()`, `levelUp()`, `useConsumable()`, `takeRest()`, and dynamic AC/HP calculation |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RulesEngine.java` | Modify | Add equipment-aware AC calculation, XP threshold lookup, and attack/damage resolution signatures |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2014RulesEngine.java` | Modify | Implement armor category AC rules, XP table, and attack calculations for 2014 rules |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2024RulesEngine.java` | Modify | Implement armor category AC rules, XP table, and attack calculations for 2024 rules |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/AddExperienceUseCase.java` | Create | Inbound port for adding XP to a character |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/LevelUpUseCase.java` | Create | Inbound port for character level-up and unrestricted multiclassing |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/RestUseCase.java` | Create | Inbound port for Short and Long Rest execution |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ManageInventoryUseCase.java` | Create | Inbound port for adding, updating (equip/attune/quantity), and removing character items |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/UseConsumableUseCase.java` | Create | Inbound port for using a consumable item from inventory |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java` | Create | Application service implementing XP addition and multi-step Level Up |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/InventoryService.java` | Create | Application service implementing inventory mutations, AC recalculation, and consumable usage |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RestService.java` | Create | Application service implementing Short and Long Rest charge and HP recovery |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydrator.java` | Modify | Hydrate inventory items, dynamic AC, unlocked feats with charges, and derived weapon/feat actions |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java` | Modify | Support creating homebrew subclasses and persisting mechanical properties on items, feats, and classes |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ItemJpaEntity.java` | Modify | Map `mechanical_properties JSONB` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntity.java` | Modify | Map `feat_type` and `mechanical_effects JSONB` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SubclassJpaEntity.java` | Create | JPA entity for `player_character.subclasses` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ArchetypeFeatJpaEntity.java` | Create | JPA entity for `player_character.archetype_feats` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SubclassFeatJpaEntity.java` | Create | JPA entity for `player_character.subclass_feats` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterItemJpaEntity.java` | Create | JPA entity for `player_character.character_items` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterFeatJpaEntity.java` | Create | JPA entity for `player_character.character_feats` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java` | Modify | Map `experience_points`, `campaign_id`, `character_items`, and `character_feats` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java` | Modify | Implement queries for subclasses, class progression tables, and mechanical JSONB mapping |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java` | Modify | Expose endpoints for XP, Level Up, Rest, Inventory CRUD, and Consumable Use |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java` | Modify | Expose endpoints for Subclasses, Class Progression, and Homebrew Subclasses |
| `player-character-service/backend/src/main/java/com/dungeoneer/core/web/GlobalExceptionHandler.java` | Modify | Add RFC 7807 `ProblemDetail` handlers for `HttpMessageNotReadableException` and `MethodArgumentTypeMismatchException` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java` | Modify | Initialize characters at Level 1 with target level's XP floor, unlock Level 1 class/subclass feats and `SRD_2024` Origin Feat, and apply initial `MechanicalEffects` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateCharacterRequest.java` | Modify | Add optional `subclassId` (for `subclassLevel == 1`) and `originFeatId` (`SRD_2024`) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateHomebrewArchetypeRequest.java` | Modify | Add `subclassLevel` and hybrid progression feature entries (inline new features + linked catalog `featId`s) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateHomebrewSubclassRequest.java` | Modify | Add hybrid progression feature entries (inline new features + linked catalog `featId`s) |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java` | Modify | Add `removeRole(Role)` domain method protecting base `Role.PLAYER` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/service/AuthService.java` | Modify | Implement `toggleDungeonMasterRole(UUID userId)` persisting role change and issuing refreshed JWT |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java` | Modify | Expose `POST /api/auth/me/roles/toggle-dm` returning `AuthResponseDto` |
| `dungeoneer-frontend/lib/hooks/use-auth.ts` | Modify | Add `toggleDM()` action updating stored JWT token and user state |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Add interactive `Mestre de Jogo (DM)` role toggle switch inside profile dropdown |
| `dungeoneer-frontend/app/profile/page.tsx` | Modify | Make `Mestre de Jogo (DUNGEON_MASTER)` toggleable on/off in "Papéis Ativos" while keeping `PLAYER` fixed |
| `dungeoneer-frontend/app/characters/new/page.tsx` | Modify | Restrict creation to primary class only, always capture `hpCalculationMethod`, show Subclass selector when `subclassLevel == 1`, and add Origin Feat selection for `SRD_2024` |
| `dungeoneer-frontend/components/encyclopedia/reference-card.tsx` | Modify | Support optional `onClick` interactive card behavior |
| `dungeoneer-frontend/components/encyclopedia/class-detail-modal.tsx` | Create | Modal rendering class proficiencies, Level 1–20 Progression Table, and expandable Subclasses |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | Add types and TanStack Query hooks for XP, Level Up, Rest, Inventory, Consumables, and Actions |
| `dungeoneer-frontend/lib/api/encyclopedia.ts` | Modify | Add mechanical types, `SubclassSummary`, `ClassProgressionEntry`, and subclass query hooks |
| `dungeoneer-frontend/lib/api/homebrew.ts` | Modify | Add mechanical payload schemas and `createHomebrewSubclass` mutation |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Modify | Integrate XP progress bar, Level Up CTA, and active `Inventário`, `Talentos`, and `Ações` tabs |
| `dungeoneer-frontend/components/character-sheet/xp-progress-bar.tsx` | Create | D&D 5e XP progress bar component with level threshold indicators |
| `dungeoneer-frontend/components/character-sheet/add-xp-modal.tsx` | Create | Modal to add XP or trigger Milestone level-up |
| `dungeoneer-frontend/components/character-sheet/level-up-wizard-modal.tsx` | Create | Multi-step Level Up & Multiclass Wizard modal |
| `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` | Create | Interactive Inventory tab with equip/attune toggles, weight tracker, and consumable use |
| `dungeoneer-frontend/components/character-sheet/feats-tab.tsx` | Create | Categorized Feats & Class Features tab with charge tracking and Short/Long Rest controls |
| `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` | Create | Combat Actions & Weapon Attacks tab with interactive dice roller |
| `dungeoneer-frontend/components/character-sheet/dice-roller-modal.tsx` | Create | Interactive dice roller modal for weapon attacks, versatile damage, and feature rolls |
| `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` | Modify | Add dynamic mechanical sub-forms for weapons, armors, consumables, feats, classes, and subclasses |
| `dungeoneer-frontend/app/encyclopedia/classes/page.tsx` | Modify | Render level 1–20 progression table and subclasses list per class |

### Data Model Changes
1. **`player_character.items`**:
   - Add `mechanical_properties JSONB NOT NULL DEFAULT '{}'::jsonb`.
2. **`player_character.feats`**:
   - Add `feat_type VARCHAR(30) NOT NULL DEFAULT 'GENERAL'` (`CHECK (feat_type IN ('GENERAL', 'ORIGIN', 'CLASS', 'SUBCLASS', 'LINEAGE'))`).
   - Add `mechanical_effects JSONB NOT NULL DEFAULT '{}'::jsonb`.
3. **`player_character.archetypes`**:
   - Add `subclass_level INT NOT NULL DEFAULT 3`.
4. **`player_character.subclasses`**:
   - Ensure columns `id UUID PK`, `archetype_id UUID FK`, `ruleset VARCHAR(20)`, `slug VARCHAR(120)`, `name_en VARCHAR(120)`, `name VARCHAR(120)`, `description TEXT`, `source VARCHAR(50)`, `owner_id UUID NULL`, `UNIQUE (ruleset, slug)`.
5. **`player_character.archetype_feats` & `player_character.subclass_feats`**:
   - Associative tables linking `archetype_id` / `subclass_id` to `feat_id` with `unlock_level INT NOT NULL CHECK (unlock_level BETWEEN 1 AND 20)`.
6. **`player_character.characters`**:
   - Add `experience_points INT NOT NULL DEFAULT 0 CHECK (experience_points >= 0)`.
   - Add `campaign_id UUID NULL`.
7. **`player_character.character_items` & `player_character.character_feats`**:
   - Ensure `character_items` tracks `id UUID PK`, `character_id UUID FK`, `item_id UUID FK`, `quantity INT NOT NULL DEFAULT 1`, `is_equipped BOOLEAN NOT NULL DEFAULT FALSE`, `is_attuned BOOLEAN NOT NULL DEFAULT FALSE`.
   - Ensure `character_feats` tracks `id UUID PK`, `character_id UUID FK`, `feat_id UUID FK`, `unlocked_at_level INT NOT NULL DEFAULT 1`, `current_charges INT NULL`, `max_charges INT NULL`, `choices JSONB NOT NULL DEFAULT '{}'::jsonb`.

## Acceptance Criteria

- [ ] **Given** a weapon item with `WeaponMechanics` (`damageRolls: [{diceFormula: "1d8", damageType: "SLASHING"}, {diceFormula: "1d6", damageType: "FIRE"}]`, `versatileDice: "1d10"`, `magicBonus: 1`, `properties: ["VERSATILE"]`), **When** a character equips the weapon and loads the character sheet, **Then** the `Ações` tab displays the weapon attack with calculated to-hit bonus (`proficiencyBonus + strMod + 1`), primary damage (`1d8+strMod+1 SLASHING + 1d6 FIRE`), and a one-click Versatile (`1d10`) roll option.
- [ ] **Given** an armor item with `ArmorMechanics` and `armorCategory` set to `LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, or `SHIELD`, **When** a character with DEX modifier `+3` equips the item in the `Inventário` tab, **Then** the character's `armorClass` automatically recalculates (`LIGHT`: `baseAc + 3`; `MEDIUM`: `baseAc + 2`; `HEAVY`: `baseAc`; `ROBE`: `baseAc + 3` while remaining unarmored-compatible; `SHIELD`: adds `+baseAc + magicBonus` to existing AC).
- [ ] **Given** a character with a consumable item (`Potion of Healing`, `effectType: "HEAL_HP"`, `diceFormula: "2d4+2"`, `quantity: 2`) and `currentHp: 5` (`maxHp: 20`), **When** the player clicks "Usar" (`POST /api/characters/{id}/items/{characterItemId}/use`), **Then** the server rolls `2d4+2`, increases `currentHp` (capped at `maxHp`), decrements `quantity` to `1` (or deletes the inventory entry if `quantity` reaches `0`), and returns the roll breakdown.
- [ ] **Given** a character at level 1 with `experiencePoints: 0` and `campaignId: null`, **When** the player adds `300` XP via `POST /api/characters/{id}/xp` (or selects Milestone level-up), **Then** `experiencePoints` becomes `300`, the character sheet displays the pulsing `Subir de Nível!` CTA, and opening the Level Up Wizard allows advancing to level 2 in either their current class or any new class without ability score >= 13 prerequisite restrictions.
- [ ] **Given** a player creating a new character in `/characters/new` with target starting level `5`, `hpCalculationMethod: "ROLLED"`, and primary class `Fighter` (or `Cleric` in `SRD_2014` with `subclassLevel == 1`, or any class in `SRD_2024` with an `ORIGIN` feat), **When** the character is created, **Then** the backend persists the character at `level: 1` with `experiencePoints: 6500`, unlocks Level 1 class/subclass/origin feats, and navigating to `/characters/[id]` automatically opens `LevelUpWizardModal` sequentially for levels `2 -> 3 -> 4 -> 5` with the HP method locked to `ROLLED`.
- [ ] **Given** a character leveling up to a class's `subclassLevel` (e.g., level 3 in `SRD_2024` or level 3 Fighter in `SRD_2014`) or an ASI level (`4, 8, 12, 16, 19`, Fighter `6, 14`, Rogue `10`), **When** the player submits the Level Up Wizard (`POST /api/characters/{id}/level-up`), **Then** the chosen Subclass is bound to their `ClassProgression`, all `CLASS` and `SUBCLASS` feats for that level are unlocked in `character_feats`, Max HP increases strictly via the character's persisted `hpCalculationMethod` (server-computed `AVERAGE`/`MAX` or verified `HpRoll` token for `ROLLED`) + CON modifier + `hpPerLevelBonus`, and the chosen ASI (`+2` or `+1/+1` up to 20) or `GENERAL`/`ORIGIN` Feat (including its `abilityBonuses` and retroactive `hpPerLevelBonus`) is applied.
- [ ] **Given** an authenticated user in `/profile` or the `Navbar` profile menu, **When** they toggle the `Mestre de Jogo (DUNGEON_MASTER)` switch on or off (`POST /api/auth/me/roles/toggle-dm`), **Then** `auth-service` updates the user's roles (preserving `PLAYER` as fixed base role) and returns a new JWT token that immediately updates UI permissions and API Gateway headers.
- [ ] **Given** a user browsing `/encyclopedia/classes`, **When** they click a class card, **Then** `ClassDetailModal` opens displaying saving throws, proficiencies, the Level 1–20 Progression Table with expandable feature descriptions/mechanics, and all Subclasses for that class.
- [ ] **Given** a character with active class/feat abilities that have `usesPerRest: 2` and `restType: "SHORT"` with `currentCharges: 0`, **When** the player executes a Short Rest (`POST /api/characters/{id}/rest` with `restType: "SHORT"`), **Then** `currentCharges` resets to `2`; and when executing a Long Rest (`LONG`), all `SHORT` and `LONG` rest charges reset to `maxCharges` and `currentHp` resets to `maxHp`.
- [ ] **Given** a user with `DUNGEON_MASTER` or `ADMIN` role opening `CreateHomebrewDialog`, **When** they create a Homebrew Weapon, Armor/Robe/Shield, Consumable, Feat, Class, or Subclass with mechanical properties, **Then** the backend validates the mechanical structure, persists the entity with `ruleset = HOMEBREW`, and exposes it in the Encyclopedia and Character Sheet catalogs.

## Defensive Security & Validation Considerations

### Input Validation
- All incoming DTOs (`AddExperienceRequest`, `LevelUpRequest`, `RestRequest`, `AddCharacterItemRequest`, `UpdateCharacterItemRequest`, `CreateHomebrewItemRequest`, `CreateHomebrewFeatRequest`, `CreateHomebrewSubclassRequest`) are validated with Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Min`, `@Max`, `@Size`, `@Valid`).
- Dice formulas in `WeaponMechanics`, `ConsumableMechanics`, and `MechanicalEffects` are strictly validated against the regex `^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$` in both domain Value Objects and DTO validators to prevent expression injection or denial-of-service via unbounded dice loops (max 20 dice per roll).
- Character attunement is enforced in the domain layer: attempting to attune more than 3 items simultaneously throws `CharacterValidationException` mapped to HTTP `422 Unprocessable Entity` (RFC 7807 `ProblemDetail`).
- Total character level is strictly capped at `20`, ability score increases via ASI are capped at `20` per ability, and XP additions require positive integers (`@Min(1)`).

### Secret Isolation
- No credentials, signing keys, or database URLs are hardcoded in migrations, domain classes, or frontend code. Database connections continue to use environment variables (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`).

### Authorization & Authentication Impact
- Character mutation endpoints (`POST /{id}/xp`, `POST /{id}/level-up`, `POST /{id}/rest`, `POST/PATCH/DELETE /{id}/items`, `POST /{id}/items/{itemId}/use`) verify character ownership (`X-Auth-User-Id` header matching `character.ownerId` when owned) in `player-character-service`.
- When `character.campaignId != null`, manual XP addition (`POST /{id}/xp`) by the player is rejected with HTTP `403 Forbidden` / `422 Unprocessable Entity` (`ProblemDetail`) because XP in active campaigns is managed exclusively by the Campaign DM.
- Homebrew creation endpoints (`POST /api/reference/homebrew/**`, including the new `/api/reference/homebrew/subclasses` endpoint) remain restricted to `DUNGEON_MASTER` and `ADMIN` roles at both the API Gateway (`RoleAuthorizationFilter`) and `ReferenceDataController`.

### Other Security Risks
- **Server-Authority over Dice Rolls & HP**: Level-up HP rolls (`HpCalculationMethod.ROLL`) must reference a server-generated, unconsumed `HpRoll` token (`rollId`) matching the target class's `HitDiceEnum` and character owner, preventing client-side tampering with HP values. Consumable dice formulas are rolled exclusively on the backend.
- **JSONB Deserialization Safety**: Jackson polymorphic or structured deserialization of `mechanical_properties` and `mechanical_effects` uses explicit records/enums without default typing (`enableDefaultTyping` is never used), eliminating Jackson gadget-chain deserialization vulnerabilities.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Calculate AC across `LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, and `SHIELD` armor categories with DEX `+3` | Unit (`Srd2014RulesEngineTest`, `Srd2024RulesEngineTest`) | Returns `baseAc+3` for Light/Robe, `baseAc+2` for Medium, `baseAc` for Heavy, and adds Shield bonus |
| TP-02 | Use healing potion (`2d4+2`) from inventory via `InventoryService` | Unit & Integration (`InventoryServiceTest`, `PlayerCharacterControllerIT`) | Rolls `2d4+2`, heals `currentHp` up to `maxHp`, decrements `quantity` (or removes row at 0) |
| TP-03 | Add XP and execute Level Up (single class & unrestricted multiclass) with subclass, HP roll/average, unlocked class feats, and ASI/Feat choice | Unit & Integration (`LevelUpServiceTest`, `PlayerCharacterControllerIT`) | Increments level, binds subclass, adds class/subclass feats, updates `maxHp` and ability scores |
| TP-04 | Execute Short Rest and Long Rest via `RestService` | Unit & Integration (`RestServiceTest`) | Short rest restores `SHORT` feat charges; Long rest restores all `SHORT`/`LONG` charges and `currentHp = maxHp` |
| TP-05 | Render `Inventário`, `Talentos`, `Ações` tabs, XP bar, and Level Up Wizard on `/characters/[id]` | Frontend Unit (`character-sheet.test.tsx`, `level-up-wizard.test.tsx`) | Tabs display items, AC updates on equip, feats group by type, actions roll weapon damage, and wizard submits level-up |
| TP-06 | Create Homebrew Weapon, Armor (`ROBE`), Consumable, Feat, and Subclass via `CreateHomebrewDialog` | Frontend Unit & Backend Integration (`homebrew-creation.test.tsx`, `HomebrewControllerIT`) | Dynamic fields validate and persist `mechanicalProperties`/`mechanicalEffects` with HTTP `201 Created` |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Attempt to attune a 4th magic item on a character | Unit & Integration | Throws `CharacterValidationException` -> HTTP `422 Unprocessable Entity` |
| TE-02 | Attempt to add XP manually when `character.campaignId != null` | Unit & Integration | Rejects request with `CharacterValidationException` -> HTTP `422 Unprocessable Entity` |
| TE-03 | Attempt to level up without sufficient XP when `milestone = false` or beyond total level 20 | Unit & Integration | Throws `CharacterValidationException` -> HTTP `422 Unprocessable Entity` |
| TE-04 | Attempt to increase an ability score above 20 via ASI during Level Up | Unit & Integration | Throws `CharacterValidationException` -> HTTP `422 Unprocessable Entity` |
| TE-05 | Submit invalid dice notation (e.g., `999d1000` or SQL/script payload) in `WeaponMechanics` or `ConsumableMechanics` | Unit & Integration | Rejected by domain/Jakarta validation with HTTP `400 Bad Request` / `422 Unprocessable Entity` |

### Test Commands
- **Run backend unit & integration tests:** `mvn clean test` (executed in `player-character-service/backend`)
- **Run backend coverage check:** `mvn clean verify` (JaCoCo gate >= 80% on `domain` + `application` packages)
- **Run frontend unit tests:** `npm run test:unit` (executed in `dungeoneer-frontend`)
- **Run frontend production build check:** `npm run build` (executed in `dungeoneer-frontend`)

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Large Flyway seed migrations (`V17`, `V18`) fail on syntax or constraint mismatches against `V10`/`V13`/`V14` slugs | Med | High | Use idempotent `INSERT ... ON CONFLICT (ruleset, slug) DO UPDATE` and `UPDATE ... WHERE slug LIKE ...` statements verified against Testcontainers PostgreSQL 15 during Phase 1 |
| AC recalculation conflicts between equipped Armor, `ROBE`, Unarmored Defense (Barbarian/Monk), and passive `acBonus` feats | Med | Med | Centralize AC resolution in `RulesEngine` with deterministic precedence: `HEAVY`/`MEDIUM`/`LIGHT` armor overrides Unarmored Defense, whereas `ROBE` counts as unarmored and stacks with `SHIELD` and `acBonus` |
| Domain layer accidentally imports Jackson or Hibernate annotations for JSONB Value Objects | Low | High | Enforce strict Clean Architecture separation: keep `ItemMechanics` and `MechanicalEffects` pure Java in `domain/model/` and perform JSON serialization/mapping inside `adapter/out/persistence/` |

## Specification Changes & Delta Stubs

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Documents Flyway migrations `V16`–`V18`, JSONB schemas for `items.mechanical_properties` and `feats.mechanical_effects`, new columns on `characters` (`experience_points`, `campaign_id`) and `archetypes` (`subclass_level`), new progression tables (`archetype_feats`, `subclass_feats`), and updated `character_items` and `character_feats` schemas.

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Documents polymorphic `ItemMechanics` (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`), `MechanicalEffects`, `Subclass` domain model, `RulesEngine` AC calculation by `ArmorCategory`, UC2 (XP & Unrestricted Multiclass Level Up), Consumable usage, Short/Long Rest recharge mechanics, and new REST endpoints.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Documents the Character Sheet XP progress bar, `+ Adicionar XP` modal, multi-step Level Up Wizard, interactive `Inventário`, `Talentos`, and `Ações` tabs with dice roller modal, dynamic mechanical forms in `CreateHomebrewDialog`, and the 1–20 progression table in `/encyclopedia/classes`.

### `specs/infrastructure-and-testing-delta.md`
- **Target Canonical Spec:** `openspec/specs/infrastructure-and-testing.md`
- **Summary of Updates:** Documents Testcontainers integration test coverage for JSONB migrations (`V16`–`V18`), unit test suites for AC/Attack/Level-Up/Consumable domain logic, `auth-service` DM role toggle tests, and Vitest test suites for the new Character Sheet tabs, Level Up Wizard, Class Detail Modal, and Homebrew forms.

### `specs/auth-service-delta.md`
- **Target Canonical Spec:** `openspec/specs/system-architecture.md`
- **Summary of Updates:** Documents `POST /api/auth/me/roles/toggle-dm` in `auth-service` allowing authenticated users to toggle `DUNGEON_MASTER` on/off while preserving `PLAYER` as the mandatory base role, returning a refreshed `AuthResponseDto` with a new signed JWT.

## References

- [prompt-mechanical-properties-and-class-progression-spec.md](file:///D:/Projects/Dungeoneer/prompt-mechanical-properties-and-class-progression-spec.md)
- [player-character-domain.md](file:///D:/Projects/Dungeoneer/openspec/specs/player-character-domain.md)
- [player-character-data-model.md](file:///D:/Projects/Dungeoneer/openspec/specs/player-character-data-model.md)
- [frontend-web-app.md](file:///D:/Projects/Dungeoneer/openspec/specs/frontend-web-app.md)
- [sdd-stack-contract.md](file:///D:/Projects/Dungeoneer/openspec/specs/sdd-stack-contract.md)
