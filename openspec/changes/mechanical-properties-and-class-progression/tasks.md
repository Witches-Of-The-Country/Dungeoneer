# Tasks: Mechanical Properties, Class Progression & Character Sheet Evolution

> **Change ID:** `mechanical-properties-and-class-progression`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### Phase 1: Database Schema Migrations & SRD (2014 & 2024) Seed Data
- [x] Create Flyway migration `V16__add_mechanical_properties_and_class_progression_tables.sql` in `player-character-service` adding `items.mechanical_properties JSONB`, `feats.feat_type VARCHAR(30)`, `feats.mechanical_effects JSONB`, `archetypes.subclass_level INT`, `characters.experience_points INT`, `characters.campaign_id UUID`, and tables/columns for `subclasses`, `archetype_feats`, `subclass_feats`, `character_items`, and `character_feats`.
- [x] Create Flyway migration `V17__seed_srd_subclasses_and_progression_feats.sql` seeding all 13 SRD Subclasses (across `SRD_2014` and `SRD_2024`), updating `archetypes.subclass_level`, and seeding level 1–20 Class Feats (`archetype_feats`) and Subclass Feats (`subclass_feats`) with `mechanical_effects`.
- [x] Create Flyway migration `V18__populate_srd_mechanical_properties.sql` populating structured `mechanical_properties` JSONB for existing SRD weapons (damage rolls, versatile dice, weapon properties), armors/robes/shields (`baseAc`, `armorCategory`, `strengthRequirement`, `stealthDisadvantage`), consumables (`Potion of Healing`, etc.), and `mechanical_effects` JSONB for existing SRD Origin/General/Epic Boon feats (e.g., *Tough*, *Alert*, *Grappler*, *War Caster*).

### Phase 2: Clean Architecture Domain Value Objects, Entities & Rules Engines
- [x] Create pure Java 21 domain enums and Value Objects in `com.dungeoneer.playerCharacter.domain.model` (zero framework imports): `DamageType`, `DamageRoll`, `WeaponProperty`, `ArmorCategory` (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), `ConsumableEffectType`, `ItemMechanics` (sealed interface), `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, `FeatType`, `RestType`, and `MechanicalEffects`.
- [x] Implement full `Subclass` domain entity in `domain/model/Subclass.java` (replacing the empty stub) and update `Archetype.java`, `Item.java`, `Feat.java`, `ClassProgression.java`, `CharacterItem.java`, and `CharacterFeat.java` with mechanical properties, invariants, and progression metadata.
- [x] Update `RulesEngine.java`, `Srd2014RulesEngine.java`, and `Srd2024RulesEngine.java` to implement equipment-aware Armor Class calculation (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`, Unarmored Defense, and passive `acBonus`), D&D 5e XP threshold table lookup (levels 1–20, 0 to 355,000 XP), ASI level threshold resolution (standard `4, 8, 12, 16, 19` + Fighter `6, 14` + Rogue `10`), and weapon attack/damage calculation.
- [x] Update `PlayerCharacter.java` with `experiencePoints`, `campaignId`, `items`, `feats`, and domain methods `addExperience()`, `levelUp()` (supporting unrestricted multiclassing without ability >= 13 prerequisite checks), `equipItem()`, `attuneItem()` (max 3 slots), `useConsumable()`, and `takeRest()` (`SHORT` and `LONG`).

### Phase 3: Persistence Adapters, Application Services & REST Controllers
- [x] Create and update JPA entities and Spring Data repositories in `adapter/out/persistence/`: `ItemJpaEntity`, `FeatJpaEntity`, `SubclassJpaEntity`, `ArchetypeFeatJpaEntity`, `SubclassFeatJpaEntity`, `CharacterItemJpaEntity`, `CharacterFeatJpaEntity`, `PlayerCharacterJpaEntity`, `ReferenceDataPersistenceAdapter`, and `PlayerCharacterPersistenceAdapter`.
- [x] Create inbound use case ports (`AddExperienceUseCase`, `LevelUpUseCase`, `RestUseCase`, `ManageInventoryUseCase`, `UseConsumableUseCase`) and implement application services (`LevelUpService`, `InventoryService`, `RestService`) and updated `CharacterSheetHydrator` and `HomebrewService`.
- [x] Expose and document REST endpoints in `PlayerCharacterController` (`POST /api/characters/{id}/xp`, `POST /api/characters/{id}/level-up`, `POST /api/characters/{id}/rest`, `POST/PATCH/DELETE /api/characters/{id}/items`, `POST /api/characters/{id}/items/{characterItemId}/use`) and `ReferenceDataController` (`GET /api/reference/subclasses`, `GET /api/reference/archetypes/{id}/progression`, `POST /api/reference/homebrew/subclasses`).

### Phase 4: Frontend API Client, XP Progress Bar & Level Up Wizard
- [x] Update `dungeoneer-frontend/lib/api/characters.ts`, `lib/api/encyclopedia.ts`, and `lib/api/homebrew.ts` with TypeScript interfaces for `ItemMechanics`, `MechanicalEffects`, `CharacterItem`, `CharacterFeat`, `CharacterAction`, `SubclassSummary`, `ClassProgressionEntry`, and TanStack Query mutation/query hooks.
- [x] Create `XpProgressBar` (`components/character-sheet/xp-progress-bar.tsx`) and `AddXpModal` (`components/character-sheet/add-xp-modal.tsx`) displaying 0–355,000 XP progression, campaign lock badge when `campaignId != null`, and pulsing `Subir de Nível!` CTA.
- [x] Create `LevelUpWizardModal` (`components/character-sheet/level-up-wizard-modal.tsx`) supporting XP or Milestone (`Marco`) advancement, class progression or unrestricted multiclass selection, Subclass choice at `subclassLevel`, HP increase (Roll, Average, or Max + CON mod + `hpPerLevelBonus`), unlocked Class/Subclass Feats preview, and ASI (+2 or +1/+1 up to 20) vs Feat selection.

### Phase 5: Character Sheet Dashboard Tabs Activation (`Inventário`, `Talentos`, `Ações`)
- [x] Create `InventoryTab` (`components/character-sheet/inventory-tab.tsx`) replacing "Em breve" with item search/addition from catalog, quantity controls, carrying weight tracker, equip/unequip toggle with immediate AC recalculation, attunement toggle (max 3 slots), and "Usar" button for consumables.
- [x] Create `FeatsTab` (`components/character-sheet/feats-tab.tsx`) replacing "Em breve" with categorized sections (`Origem & Gerais`, `Características de Classe`, `Características de Subclasse`, `Traços de Linhagem`), interactive charge checkboxes/buttons, and `Descanso Curto` / `Descanso Longo` recovery buttons.
- [x] Create `ActionsTab` (`components/character-sheet/actions-tab.tsx`) and `DiceRollerModal` (`components/character-sheet/dice-roller-modal.tsx`) replacing "Em breve" with auto-generated weapon attacks (to-hit roll, primary/secondary damage rolls, Versatile two-handed option) and active class/feat actions.
- [x] Integrate `XpProgressBar`, `LevelUpWizardModal`, `InventoryTab`, `FeatsTab`, and `ActionsTab` into `dungeoneer-frontend/app/characters/[id]/page.tsx`.

### Phase 6: Dynamic Homebrew Creation Dialog & Class Encyclopedia Progression
- [x] Update `CreateHomebrewDialog` (`components/homebrew/create-homebrew-dialog.tsx`) with dynamic mechanical sub-forms for Weapons (damage rolls array, versatile die, magic bonus, weapon properties), Armor (`baseAc`, `armorCategory` including `ROBE` and `SHIELD`, STR requirement, stealth disadvantage), Consumables (`effectType`, `diceFormula`), Feats (`featType`, passive bonuses, active actions/charges), Classes (`subclassLevel` and level 1–20 feats), and Subclasses.
- [x] Update `/encyclopedia/classes` (`app/encyclopedia/classes/page.tsx`) to render the complete level 1–20 progression table (`Bônus de Proficiência`, `Características de Classe`) and expandable Subclass cards with their level-gated features.

## Security Validation Tasks

- [x] Validate all incoming request DTOs (`AddExperienceRequest`, `LevelUpRequest`, `RestRequest`, `AddCharacterItemRequest`, `UpdateCharacterItemRequest`, `CreateHomebrewItemRequest`, `CreateHomebrewFeatRequest`, `CreateHomebrewSubclassRequest`) with Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Min`, `@Max`, `@Pattern`, `@Valid`).
- [x] Enforce server-side dice notation regex validation (`^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$`), max 3 attuned items invariant, campaign XP lock (`campaignId != null`), and server-validated `HpRoll` token verification on level-up HP rolls.
- [x] Enforce role-based access control (`DUNGEON_MASTER` and `ADMIN`) on `POST /api/reference/homebrew/subclasses` and character ownership checks on all character progression/inventory endpoints.

## Test Tasks

- [x] Write backend unit tests (`Srd2014RulesEngineTest`, `Srd2024RulesEngineTest`, `ItemMechanicsTest`, `MechanicalEffectsTest`, `LevelUpServiceTest`, `InventoryServiceTest`, `RestServiceTest`) covering AC calculation for `LIGHT`/`MEDIUM`/`HEAVY`/`ROBE`/`SHIELD`, consumable usage and auto-removal at quantity 0, unrestricted multiclassing, ASI vs Feat branching, and Short/Long Rest recharge.
- [x] Write backend Testcontainers integration tests (`PlayerCharacterProgressionIT`, `ReferenceDataProgressionIT`) verifying Flyway migrations `V16`–`V18`, JSONB serialization/deserialization, level-up persistence, inventory equip/consumable endpoints, and homebrew subclass creation.
- [x] Write frontend Vitest unit tests (`__tests__/character-sheet.test.tsx`, `__tests__/level-up-wizard.test.tsx`, `__tests__/homebrew-creation.test.tsx`, `__tests__/encyclopedia-browse.test.tsx`) verifying the activated `Inventário`, `Talentos`, and `Ações` tabs, XP bar & Level Up Wizard flows, dynamic Homebrew mechanical inputs, and Class Encyclopedia 1–20 progression table.
- [x] Run `mvn clean verify` in `player-character-service/backend` (confirming >= 80% JaCoCo line coverage) and `npm run test:unit && npm run build` in `dungeoneer-frontend`.

## Documentation Tasks

- [x] Verify OpenAPI 3.0 annotations (`@Operation`, `@ApiResponse`, `@Schema`) on all new and updated endpoints in `PlayerCharacterController` and `ReferenceDataController`.
- [x] Ensure all 4 delta specification files in `openspec/changes/mechanical-properties-and-class-progression/specs/` accurately reflect implemented schemas, endpoints, and UI contracts.

## Review Gate

- [x] All implementation tasks complete
- [x] All security validation tasks complete
- [x] All tests passing (`mvn clean verify` and `npm run test:unit && npm run build`)
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
