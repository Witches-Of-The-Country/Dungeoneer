# Phased Execution Prompt: Mechanical Properties, Class Progression & Character Sheet Evolution

Focus: Full-Stack / Spring Boot 4 (Java 21) / PostgreSQL 15 / Clean Architecture / Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Execute Full Phased Implementation of Change `mechanical-properties-and-class-progression`

## Global Execution Directives
- **Active Change Plan:** `openspec/changes/mechanical-properties-and-class-progression/tasks.md`
- **Approved Proposal:** `openspec/changes/mechanical-properties-and-class-progression/proposal.md`
- **Delta Specs:**
  - `openspec/changes/mechanical-properties-and-class-progression/specs/player-character-data-model-delta.md`
  - `openspec/changes/mechanical-properties-and-class-progression/specs/player-character-domain-delta.md`
  - `openspec/changes/mechanical-properties-and-class-progression/specs/frontend-web-app-delta.md`
  - `openspec/changes/mechanical-properties-and-class-progression/specs/infrastructure-and-testing-delta.md`
- **Cumulative Walkthrough Artifact:** `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` (maintained strictly in Portuguese `pt-BR`).
- **Architectural & Stack Invariants:**
  - **Clean Architecture (Hexagonal):** Zero framework imports in backend `domain/model/` (strictly NO Spring, JPA, Hibernate, or Lombok imports on domain classes). All JSONB serialization and mapping must reside strictly within `adapter/out/persistence/`.
  - **Build & Test Tools:** ALWAYS use global `mvn` (e.g., `mvn -f player-character-service/backend/pom.xml ...`), NEVER maven wrapper (`mvnw`). Backend line coverage must reach `>= 80%` on `domain` and `application` packages via JaCoCo (`mvn clean verify`).
  - **Error Handling:** Return RFC 7807 `ProblemDetail` responses for all REST errors via `@ControllerAdvice`.
  - **Sequential Phase Execution & Phase Pauses:** Execute exactly one phase at a time. Run local verification, mark completed tasks as `[x]` in `tasks.md`, provide the ready-to-use Conventional Commit command, update the cumulative walkthrough in `pt-BR`, and pause for user approval before advancing to the next phase.

---

### Phase 1: Database Schema Migrations & SRD (2014 & 2024) Seed Data

Focus: Backend / PostgreSQL 15 / Flyway Migrations / Relational & JSONB Data Modeling
Task: Execute Phase 1: Database Schema Migrations V16, V17, and V18

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 1)
- Delta Specs: `specs/player-character-data-model-delta.md`
- Scope: Execute Phase 1 only. Implement Flyway migrations V16, V17, and V18 in `player-character-service/backend/src/main/resources/db/migration/`. Do not begin domain or application layer implementation.

#### Implementation Steps
1. **Flyway Migration `V16__add_mechanical_properties_and_class_progression_tables.sql`:**
   - In `player_character.items`, add `mechanical_properties JSONB NOT NULL DEFAULT '{}'::jsonb` and create GIN index `idx_items_mechanical_properties`.
   - In `player_character.feats`, add `feat_type VARCHAR(30) NOT NULL DEFAULT 'GENERAL'` with check constraint `('GENERAL', 'ORIGIN', 'CLASS', 'SUBCLASS', 'LINEAGE')`, index `idx_feats_feat_type`, and `mechanical_effects JSONB NOT NULL DEFAULT '{}'::jsonb`.
   - In `player_character.archetypes`, add `subclass_level INT NOT NULL DEFAULT 3 CHECK (subclass_level BETWEEN 1 AND 20)`.
   - In `player_character.subclasses`, add `slug VARCHAR(120)`, `name_en VARCHAR(120)`, `description TEXT NOT NULL DEFAULT ''`, `source VARCHAR(50) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, and unique constraint/index `(ruleset, slug)`.
   - Create associative progression tables:
     - `player_character.archetype_feats (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id) ON DELETE CASCADE, feat_id UUID NOT NULL REFERENCES player_character.feats(id) ON DELETE CASCADE, unlock_level INT NOT NULL CHECK (unlock_level BETWEEN 1 AND 20), UNIQUE(archetype_id, feat_id, unlock_level))`.
     - `player_character.subclass_feats (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), subclass_id UUID NOT NULL REFERENCES player_character.subclasses(id) ON DELETE CASCADE, feat_id UUID NOT NULL REFERENCES player_character.feats(id) ON DELETE CASCADE, unlock_level INT NOT NULL CHECK (unlock_level BETWEEN 1 AND 20), UNIQUE(subclass_id, feat_id, unlock_level))`.
   - In `player_character.characters`, add `experience_points INT NOT NULL DEFAULT 0 CHECK (experience_points >= 0)` and `campaign_id UUID NULL`.
   - In `player_character.character_items`, ensure columns `quantity INT NOT NULL DEFAULT 1`, `is_equipped BOOLEAN NOT NULL DEFAULT FALSE`, `is_attuned BOOLEAN NOT NULL DEFAULT FALSE` exist.
   - In `player_character.character_feats`, ensure columns `unlocked_at_level INT NOT NULL DEFAULT 1`, `current_charges INT NULL`, `max_charges INT NULL`, and `choices JSONB NOT NULL DEFAULT '{}'::jsonb` exist.
2. **Flyway Migration `V17__seed_srd_subclasses_and_progression_feats.sql`:**
   - Seed all 13 SRD Subclasses for both `SRD_2014` and `SRD_2024` using idempotent `INSERT ... ON CONFLICT (ruleset, slug) DO UPDATE`:
     - Barbarian: *Path of the Berserker* (`Caminho do Furioso`)
     - Bard: *College of Lore* (`Colégio do Conhecimento`)
     - Cleric: *Life Domain* (`Domínio da Vida`)
     - Druid: *Circle of the Land* (`Círculo da Terra`)
     - Fighter: *Champion* (`Campeão`)
     - Monk: *Way of the Open Hand* (`Caminho da Mão Aberta`)
     - Paladin: *Oath of Devotion* (`Juramento de Devoção`)
     - Ranger: *Hunter* (`Caçador`)
     - Rogue: *Thief* (`Ladrão`)
     - Sorcerer: *Draconic Bloodline* (`Linhagem Dracônica`)
     - Warlock: *The Fiend* (`O Corruptor`)
     - Wizard: *School of Evocation* (`Escola de Evocação`)
     - Artificer (where present): *Alchemist* (`Alquimista`)
   - Update `archetypes.subclass_level` per ruleset convention (standard level 3 for `SRD_2024`; class-specific for `SRD_2014`: Cleric 1, Druid 2, Wizard 2, Sorcerer 1, Warlock 1, others 3).
   - Seed core level 1–20 Class Feats (`archetype_feats`) and Subclass Feats (`subclass_feats`) with `mechanical_effects` JSONB (e.g., *Second Wind*, *Action Surge*, *Rage*, *Unarmored Defense*, *Sneak Attack*, *Cunning Action*, *Divine Smite*, *Channel Divinity*, *Wild Shape*, etc.).
3. **Flyway Migration `V18__populate_srd_mechanical_properties.sql`:**
   - Populate structured `mechanical_properties` JSONB for SRD weapons (composite damage rolls, versatile dice, magicBonus, properties), armors/robes/shields (`armorCategory`, `baseAc`, `magicBonus`, `strengthRequirement`, `stealthDisadvantage`), and consumables (`Potion of Healing` `2d4+2` `HEAL_HP`, etc.).
   - Populate `feat_type` (`ORIGIN`, `GENERAL`) and `mechanical_effects` on existing SRD feats (e.g., *Tough* with `hpPerLevelBonus: 2`, *Alert* with `initiativeBonus`, *Grappler*, *War Caster*).
4. Update `openspec/changes/mechanical-properties-and-class-progression/tasks.md` marking Phase 1 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test-compile
  ```
- Commit locally:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/V16__add_mechanical_properties_and_class_progression_tables.sql \
          player-character-service/backend/src/main/resources/db/migration/V17__seed_srd_subclasses_and_progression_feats.sql \
          player-character-service/backend/src/main/resources/db/migration/V18__populate_srd_mechanical_properties.sql \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(backend): add Flyway migrations V16-V18 for mechanical properties, subclasses, and class progression"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update or create the shared walkthrough artifact (`openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR):
1. List of migration files created and schemas updated.
2. Verification results of SQL compilation/validation.
3. Handoff context prepared for Phase 2 (Clean Architecture Domain Value Objects, Entities & Rules Engines).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 2.*

---

### Phase 2: Clean Architecture Domain Value Objects, Entities & Rules Engines

Focus: Backend / Java 21 / Clean Architecture Domain Modeling (Zero Framework Imports)
Task: Execute Phase 2: Domain Item Mechanics, Unified Feat Effects, Subclass Entity, and Dynamic RulesEngine

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 2)
- Delta Specs: `specs/player-character-domain-delta.md`
- Scope: Execute Phase 2 only. Implement pure Java 21 domain Value Objects, enums, Subclass entity, and RulesEngine calculations in `com.dungeoneer.playerCharacter.domain.model`. Zero framework imports allowed.

#### Implementation Steps
1. **Pure Domain Enums and Value Objects (`domain/model/`):**
   - Create `DamageType` enum (`SLASHING`, `PIERCING`, `BLUDGEONING`, `FIRE`, `COLD`, `LIGHTNING`, `THUNDER`, `POISON`, `ACID`, `PSYCHIC`, `NECROTIC`, `RADIANT`, `FORCE`).
   - Create `DamageRoll` record/VO (`String diceFormula`, `DamageType damageType`) validating dice regex `^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$`.
   - Create `WeaponProperty` enum (`FINESSE`, `LIGHT`, `HEAVY`, `REACH`, `TWO_HANDED`, `THROWN`, `AMMUNITION`, `LOADING`, `VERSATILE`).
   - Create `ArmorCategory` enum (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`).
   - Create `ConsumableEffectType` enum (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`).
   - Create `ItemMechanics` sealed interface permitting:
     - `WeaponMechanics` (`List<DamageRoll> damageRolls`, `String versatileDice`, `int magicBonus`, `List<WeaponProperty> properties`).
     - `ArmorMechanics` (`ArmorCategory armorCategory`, `int baseAc`, `int magicBonus`, `int strengthRequirement`, `boolean stealthDisadvantage`).
     - `ConsumableMechanics` (`ConsumableEffectType effectType`, `String diceFormula`, `String effectDescription`).
     - `EmptyItemMechanics` (fallback VO for non-mechanical items).
   - Create `FeatType` enum (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`).
   - Create `RestType` enum (`SHORT`, `LONG`, `NONE`).
   - Create `MechanicalEffects` record/VO (`Map<String, Integer> abilityBonuses`, `int acBonus`, `int initiativeBonus`, `int speedBonus`, `int hpPerLevelBonus`, `List<String> grantedProficiencies`, `String actionName`, `String actionType`, `String diceFormula`, `Integer usesPerRest`, `RestType restType`).
   - Create `CharacterItem` domain model (`UUID id`, `UUID itemId`, `int quantity`, `boolean equipped`, `boolean attuned`).
   - Create `CharacterFeat` domain model (`UUID id`, `UUID featId`, `int unlockedAtLevel`, `Integer currentCharges`, `Integer maxCharges`, `String choicesJson`).
2. **Subclass Domain Entity & Entity Updates:**
   - Implement full `Subclass` domain entity in `domain/model/Subclass.java` (replacing stub) linked to parent `Archetype` with `ruleset`, `slug`, `nameEn`, `name`, `description`, `source`, and `ownerId`.
   - Update `Archetype.java` with `subclassLevel` and helper methods for class ASI levels (`4, 8, 12, 16, 19`, plus Fighter `6, 14` and Rogue `10`).
   - Update `Item.java` with `ItemMechanics mechanicalProperties`.
   - Update `Feat.java` with `FeatType featType` and `MechanicalEffects mechanicalEffects`.
   - Update `ClassProgression.java` to support `subclassId` binding.
   - Update `PlayerCharacter.java` with `experiencePoints`, `campaignId`, `items`, `feats`, and domain methods:
     - `addExperience(int amount)`
     - `levelUp(...)` (unrestricted multiclassing without ability score >= 13 prerequisite restrictions)
     - `equipItem(UUID itemId, boolean equip)` and `attuneItem(UUID itemId, boolean attune)` (enforcing max 3 attuned items invariant)
     - `useConsumable(UUID characterItemId)`
     - `takeRest(RestType restType)` (`SHORT` and `LONG`)
3. **RulesEngine AC and Progression Calculations:**
   - In `RulesEngine`, `Srd2014RulesEngine`, and `Srd2024RulesEngine`:
     - Implement dynamic `calculateArmorClass`:
       - `HEAVY`: `baseAc + magicBonus` (cancels Unarmored Defense).
       - `MEDIUM`: `baseAc + magicBonus + min(dexMod, 2)` (cancels Unarmored Defense).
       - `LIGHT`: `baseAc + magicBonus + dexMod` (cancels Unarmored Defense).
       - `ROBE` or unarmored: `baseAc + magicBonus + dexMod` (or Unarmored Defense Barbarian `10 + dexMod + conMod` / Monk `10 + dexMod + wisMod`), where robe magic bonus stacks additively.
       - Add equipped `SHIELD` bonus (`baseAc + magicBonus`).
       - Add passive `acBonus` from unlocked feats and attuned items.
     - Implement D&D 5e XP threshold table lookup (levels 1–20, 0 to 355,000 XP).
     - Implement weapon attack roll bonus (`abilityMod + proficiencyBonus + magicBonus`, resolving STR or DEX for `FINESSE`/ranged) and damage roll formulas.
4. **Domain Unit Tests:**
   - Create unit tests covering `ItemMechanicsTest`, `MechanicalEffectsTest`, `Srd2014RulesEngineTest`, `Srd2024RulesEngineTest`, and `PlayerCharacterTest`.
5. Update `tasks.md` marking Phase 2 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test -Dtest=*RulesEngineTest,*ItemMechanicsTest,*MechanicalEffectsTest,*PlayerCharacterTest
  ```
- Commit locally:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/ \
          player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/ \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(backend): implement domain item mechanics, mechanical effects, subclass entity, and rules engine AC calculations"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR:
1. List of domain Value Objects, entities, and tests implemented.
2. Unit test results verifying zero-framework domain models and AC calculations.
3. Handoff context prepared for Phase 3 (Persistence Adapters, Application Services & REST Controllers).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 3.*

---

### Phase 3: Persistence Adapters, Application Services & REST Controllers

Focus: Backend / Spring Boot 4 / Hexagonal Architecture / REST APIs / Testcontainers
Task: Execute Phase 3: JPA Entities, LevelUpService, InventoryService, RestService, and REST Controllers

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 3)
- Delta Specs: `specs/player-character-data-model-delta.md` and `specs/player-character-domain-delta.md`
- Scope: Execute Phase 3 only. Implement JPA entities, JSONB mappers, inbound use case ports, application services, and REST controllers.

#### Implementation Steps
1. **JPA Entities & Persistence Adapters (`adapter/out/persistence/`):**
   - Update `ItemJpaEntity` mapping `mechanical_properties` JSONB via Hibernate `@JdbcTypeCode(SqlTypes.JSON)`.
   - Update `FeatJpaEntity` mapping `feat_type` and `mechanical_effects` JSONB via `@JdbcTypeCode(SqlTypes.JSON)`.
   - Create `SubclassJpaEntity`, `ArchetypeFeatJpaEntity`, `SubclassFeatJpaEntity`, `CharacterItemJpaEntity`, and `CharacterFeatJpaEntity`.
   - Update `PlayerCharacterJpaEntity` mapping `experience_points`, `campaign_id`, and relationships for character items and feats.
   - Update `ReferenceDataPersistenceAdapter` and repositories to query subclasses, progression feats, and homebrew persistence.
   - Update `PlayerCharacterPersistenceAdapter` to handle inventory mutations, feats, and class progressions.
2. **Inbound Ports & Application Services:**
   - Create inbound ports in `application/port/in/`: `AddExperienceUseCase`, `LevelUpUseCase`, `RestUseCase`, `ManageInventoryUseCase`, `UseConsumableUseCase`, and their command DTOs with Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Min`, `@Max`, `@Pattern`, `@Valid`).
   - Create `LevelUpService` implementing UC2:
     - Validate XP threshold or `milestone = true` (setting XP to new level floor).
     - Support advancing existing class or adding new class (**unrestricted multiclassing without ability score >= 13 checks**).
     - Enforce Subclass choice when class reaches `subclassLevel`.
     - Resolve HP increase: validate server `HpRoll` token for `ROLL`, or apply `AVERAGE` or `MAX` + CON mod + feat `hpPerLevelBonus`.
     - Automatically unlock `archetype_feats` and `subclass_feats` for the new class level into `character_feats`.
     - Apply ASI (+2 or +1/+1 up to 20) OR add chosen Feat at class ASI levels (`4, 8, 12, 16, 19`, Fighter `6, 14`, Rogue `10`).
   - Create `InventoryService`: add/update/remove items, toggle equip (triggering immediate AC recalculation), toggle attune (max 3 slots check), use consumable (server-side roll, heal/temp HP application, decrement quantity or delete row when 0).
   - Create `RestService`: execute `SHORT` rest (recover `SHORT` feat charges) and `LONG` rest (recover all charges, restore HP to max).
   - Update `CharacterSheetHydrator`: hydrate equipped weapons with calculated attack bonus and damage formulas, armor with dynamic AC, unlocked feats with charge states, and inventory weight/attunement.
   - Update `HomebrewService`: support creating homebrew subclasses (`POST /api/reference/homebrew/subclasses`) and mechanical items/feats.
3. **Web Adapters & REST Endpoints (`adapter/in/web/`):**
   - Update `PlayerCharacterController`:
     - `POST /api/characters/{id}/xp`: add XP (reject with HTTP 422 if `campaignId != null`).
     - `POST /api/characters/{id}/level-up`: execute level-up.
     - `POST /api/characters/{id}/rest`: execute Short or Long rest.
     - `POST /api/characters/{id}/items`: add item to inventory.
     - `PATCH /api/characters/{id}/items/{characterItemId}`: equip/unequip, attune, or update quantity.
     - `DELETE /api/characters/{id}/items/{characterItemId}`: remove item from inventory.
     - `POST /api/characters/{id}/items/{characterItemId}/use`: use consumable.
     - `POST /api/characters/{id}/feats/{characterFeatId}/use`: expend feat charge.
   - Update `ReferenceDataController`:
     - `GET /api/reference/subclasses`: query subclasses by ruleset and archetype.
     - `GET /api/reference/archetypes/{id}/progression`: return 1–20 progression table and feats.
     - `POST /api/reference/homebrew/subclasses`: create homebrew subclass (secured for DM/Admin).
4. **Backend Automated Tests:**
   - Service unit tests: `LevelUpServiceTest`, `InventoryServiceTest`, `RestServiceTest`.
   - Integration tests: `PlayerCharacterProgressionIT` and `ReferenceDataProgressionIT` using Testcontainers PostgreSQL 15, verifying migrations V16–V18, JSONB serialization, level-up persistence, inventory equip/consumable endpoints, and homebrew subclass creation.
5. Update `tasks.md` marking Phase 3 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/ \
          player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/ \
          player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/ \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(backend): implement LevelUpService, InventoryService, RestService, JPA entities, and REST controllers"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR:
1. List of application services, ports, adapters, and REST endpoints created.
2. Verification results of unit and integration test executions.
3. Handoff context prepared for Phase 4 (Frontend API Client, XP Progress Bar & Level Up Wizard).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 4.*

---

### Phase 4: Frontend API Client, XP Progress Bar & Level Up Wizard

Focus: Frontend / Next.js 16 / TypeScript / TanStack Query / TailwindCSS 4
Task: Execute Phase 4: Frontend API Types, XP Progress Bar, AddXpModal, and Level Up Wizard Modal

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 4)
- Delta Specs: `specs/frontend-web-app-delta.md`
- Scope: Execute Phase 4 only. Implement TypeScript interfaces, TanStack Query hooks, `XpProgressBar`, `AddXpModal`, and `LevelUpWizardModal`. Do not begin inventory/feats/actions sheet tab refactoring.

#### Implementation Steps
1. **API Client & Type Definitions (`lib/api/characters.ts`, `lib/api/encyclopedia.ts`, `lib/api/homebrew.ts`):**
   - Define TypeScript interfaces: `ItemMechanics` (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`), `MechanicalEffects`, `CharacterItem`, `CharacterFeat`, `CharacterAction`, `SubclassSummary`, `ClassProgressionEntry`, `LevelUpRequest`, `AddExperienceRequest`, `UseConsumableResponse`.
   - Implement TanStack Query mutation and query hooks: `useAddExperience`, `useLevelUp`, `useRest`, `useAddCharacterItem`, `useUpdateCharacterItem`, `useRemoveCharacterItem`, `useUseConsumable`, `useUseFeatCharge`, `useSubclasses`, `useClassProgression`.
2. **`XpProgressBar` & `AddXpModal` (`components/character-sheet/`):**
   - Create `xp-progress-bar.tsx` rendering the 0–355,000 XP progression with Arcane Gold (`#C8AA6E`) styling, level indicators, a "+ Adicionar XP" button (disabled with tooltip when `campaignId != null`), and a pulsing "Subir de Nível!" button when XP reaches the next threshold.
   - Create `add-xp-modal.tsx` supporting XP addition or selecting "Evoluir por Marco (Milestone)" to trigger level-up without prior XP.
3. **`LevelUpWizardModal` (`components/character-sheet/level-up-wizard-modal.tsx`):**
   - Multi-step modal:
     - **Step 1 — Classe & Multiclasse**: choose to advance an existing class or select any new class from ruleset (**unrestricted multiclassing without ability score prerequisite locks**).
     - **Step 2 — Subclasse (Conditional)**: displayed when class reaches `subclassLevel`, listing available subclasses.
     - **Step 3 — Pontos de Vida (PV)**: choose between Média Fixa, Máximo, or Rolar Dado de Vida (`POST /api/characters/rolls/hp` server roll token) + CON mod + feat HP bonus.
     - **Step 4 — Características & ASI / Talento**: preview unlocked Class/Subclass feats; if class ASI level, toggle between ASI (+2 or +1/+1, max 20) or Feat selection.
   - On submit, execute `useLevelUp` mutation and invalidate character sheet cache.
4. **Frontend Unit Tests:**
   - Create `__tests__/level-up-wizard.test.tsx` verifying XP bar thresholds, milestone mode, unrestricted multiclass selection, ASI vs Feat branching, and mutation triggers.
5. Update `tasks.md` marking Phase 4 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit -- level-up-wizard
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/lib/api/ \
          dungeoneer-frontend/components/character-sheet/xp-progress-bar.tsx \
          dungeoneer-frontend/components/character-sheet/add-xp-modal.tsx \
          dungeoneer-frontend/components/character-sheet/level-up-wizard-modal.tsx \
          dungeoneer-frontend/__tests__/ \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(frontend): implement XP progress bar, AddXpModal, LevelUpWizardModal, and character progression API hooks"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR:
1. List of frontend components, API types, hooks, and tests implemented.
2. Unit test execution results for the Level Up Wizard and XP Bar.
3. Handoff context prepared for Phase 5 (Character Sheet Dashboard Tabs Activation).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 5.*

---

### Phase 5: Character Sheet Dashboard Tabs Activation (`Inventário`, `Talentos`, `Ações`)

Focus: Frontend / Next.js 16 / React 19 / Character Sheet UI
Task: Execute Phase 5: Interactive InventoryTab, FeatsTab, ActionsTab, and DiceRollerModal

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 5)
- Delta Specs: `specs/frontend-web-app-delta.md`
- Scope: Execute Phase 5 only. Replace "Em breve" placeholders in `/characters/[id]` with interactive `InventoryTab`, `FeatsTab`, `ActionsTab`, and `DiceRollerModal`.

#### Implementation Steps
1. **`InventoryTab` (`components/character-sheet/inventory-tab.tsx`):**
   - Replace "Em breve" placeholder.
   - Header: total carrying weight tracker, attunement slots counter (`Sintonização: X / 3`).
   - "Adicionar Item" dialog: searchable catalog filtered by category (`Arma`, `Armadura`, `Consumível`, `Mágico`, `Equipamento`).
   - Item cards/rows: display mechanical badges (`1d8 Cortante | Versátil (1d10)`, `CA 14 + DES (máx 2)`, `CA 12 + DES | Veste (Robe)`, `+2 CA | Escudo`, `Cura 2d4+2 PV`).
   - Interactive actions: Equip/Unequip toggle, Attune/Unattune toggle (blocked at 3/3), Quantity increment/decrement, Delete item, and "Usar" button for consumables (displaying toast with server roll result and HP change).
2. **`FeatsTab` (`components/character-sheet/feats-tab.tsx`):**
   - Replace "Em breve" placeholder.
   - Header: Short Rest ("Descanso Curto") and Long Rest ("Descanso Longo") action buttons.
   - Categorized sections: `Talentos de Origem & Gerais`, `Características de Classe`, `Características de Subclasse`, and `Traços de Linhagem`.
   - Features with charges: render interactive charge slots/dots, "Usar Habilidade" button triggering charge deduction and optional dice roll.
3. **`ActionsTab` (`components/character-sheet/actions-tab.tsx`) & `DiceRollerModal` (`components/character-sheet/dice-roller-modal.tsx`):**
   - Replace "Em breve" placeholder.
   - Weapon Attacks section: auto-generated cards for equipped weapons showing Attack Bonus (`+X para acertar`), primary and secondary damage dice, and action buttons (`Rolar Ataque`, `Rolar Dano`, `Rolar Versátil`).
   - Class & Feat Actions section: active abilities from unlocked features (e.g., *Second Wind*, *Rage*, *Sneak Attack*).
   - `DiceRollerModal`: interactive visual dice roller displaying individual dice results, modifiers, and total damage/healing.
4. **Integration in `app/characters/[id]/page.tsx`:**
   - Embed `XpProgressBar`, `LevelUpWizardModal`, `InventoryTab`, `FeatsTab`, and `ActionsTab` seamlessly into character sheet tabs.
5. **Frontend Unit Tests:**
   - Update `__tests__/character-sheet.test.tsx` verifying item equip/unequip, attunement limit, consumable usage, feat charge expenditure, rest recovery, and weapon attack rolls.
6. Update `tasks.md` marking Phase 5 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit -- character-sheet
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/components/character-sheet/ \
          dungeoneer-frontend/app/characters/\[id\]/page.tsx \
          dungeoneer-frontend/__tests__/ \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(frontend): activate character sheet Inventory, Feats, and Actions tabs with dice roller modal"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR:
1. List of sheet tab components, dice roller modal, and tests updated.
2. Unit test execution results for the character sheet tabs.
3. Handoff context prepared for Phase 6 (Dynamic Homebrew Creation Dialog & Class Encyclopedia Progression).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 6.*

---

### Phase 6: Dynamic Homebrew Creation Dialog & Class Encyclopedia Progression

Focus: Frontend / Next.js 16 / Homebrew & Encyclopedia UI
Task: Execute Phase 6: Dynamic Homebrew Mechanical Forms and Class Progression Encyclopedia Table

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Phase 6)
- Delta Specs: `specs/frontend-web-app-delta.md`
- Scope: Execute Phase 6 only. Implement dynamic mechanical forms in `CreateHomebrewDialog` and the 1–20 class progression table with subclasses in `/encyclopedia/classes`.

#### Implementation Steps
1. **`CreateHomebrewDialog` (`components/homebrew/create-homebrew-dialog.tsx`):**
   - Add 6th tab for **Subclasse** (linked to parent Archetype with level-gated subclass features).
   - Dynamic mechanical fields based on selection:
     - Item `WEAPON`: dynamic damage rolls array (`diceFormula` + `damageType`), optional `versatileDice`, `magicBonus`, `WeaponProperty` multi-select chips.
     - Item `ARMOR`: `armorCategory` selector (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), `baseAc`, `magicBonus`, `strengthRequirement`, `stealthDisadvantage`.
     - Item `CONSUMABLE`: `effectType` selector, `diceFormula`, effect description.
     - Feat: `featType`, passive bonuses (`acBonus`, `initiativeBonus`, `speedBonus`, `hpPerLevelBonus`), ability score increases, active action (`actionName`, `diceFormula`, `usesPerRest`, `restType`).
     - Class (`Archetype`): `subclassLevel` input, level 1–20 feature progression builder.
2. **`/encyclopedia/classes` (`app/encyclopedia/classes/page.tsx`):**
   - Render full level 1–20 progression table (`Nível`, `Bônus de Proficiência`, `Características de Classe`).
   - Subclasses section: expandable cards with level-by-level subclass features and descriptions.
3. **Frontend Unit Tests:**
   - Create or update `__tests__/homebrew-creation.test.tsx` and `__tests__/encyclopedia-browse.test.tsx` testing the dynamic mechanical forms and progression table rendering.
4. Update `tasks.md` marking Phase 6 tasks as `[x]`.

#### Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit -- homebrew-creation encyclopedia-browse
  ```
- Commit locally:
  ```bash
  git add dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx \
          dungeoneer-frontend/app/encyclopedia/classes/page.tsx \
          dungeoneer-frontend/__tests__/ \
          openspec/changes/mechanical-properties-and-class-progression/tasks.md
  git commit -m "feat(frontend): add dynamic mechanical homebrew forms and level 1-20 class progression encyclopedia view"
  ```

#### Phase Handoff & Cumulative Walkthrough
Update `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in pt-BR:
1. List of homebrew and encyclopedia components and tests updated.
2. Unit test execution results for the homebrew creation dialog and encyclopedia view.
3. Handoff context prepared for Phase 7 (Security Validation, Full Test Suite Verification, Spec Alignment & Final Review Gate).
*Pause execution, prompt user with the commit command, and await approval before starting Phase 7.*

---

### Phase 7: Security Validation, Full Test Suite Verification, Spec Alignment & Final Review Gate

Focus: Full-Stack / Quality Assurance / Defensive Security & SDD Archival Readiness
Task: Execute Phase 7: Pre-Archive Quality Gate, Security Auditing, Full Regression, and Final Consolidated Walkthrough

#### Execution Context
- Active Plan: `openspec/changes/mechanical-properties-and-class-progression/tasks.md` (Security Validation, Test Tasks, Documentation Tasks, and Review Gate)
- Linked Proposal: `openspec/changes/mechanical-properties-and-class-progression/proposal.md`
- Scope: Execute Phase 7 only. Run complete workspace test suites and production builds, verify defensive security considerations, ensure delta specs alignment, complete the cumulative walkthrough, and mark all tasks done.

#### Implementation Steps
1. **Security & Validation Auditing:**
   - Verify Jakarta validation on all incoming DTOs (`AddExperienceRequest`, `LevelUpRequest`, `RestRequest`, `AddCharacterItemRequest`, `UpdateCharacterItemRequest`, `CreateHomebrewItemRequest`, `CreateHomebrewFeatRequest`, `CreateHomebrewSubclassRequest`).
   - Verify server-side regex validation for dice notation (`^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$`).
   - Verify campaign XP lock rejection (`POST /api/characters/{id}/xp` returns HTTP 422 if `campaignId != null`).
   - Verify max 3 attuned items invariant enforcement.
   - Verify server-validated `HpRoll` token verification on level-up HP rolls.
   - Verify role check (`DUNGEON_MASTER` / `ADMIN`) on `POST /api/reference/homebrew/subclasses`.
   - Verify character ownership check on character progression/inventory endpoints.
2. **Full Verification Commands:**
   - Run complete backend test suite and JaCoCo coverage check:
     ```bash
     mvn -f player-character-service/backend/pom.xml clean verify
     ```
     Ensure JaCoCo coverage meets or exceeds the 80% line coverage threshold across `domain` and `application` packages.
   - Run complete frontend unit tests:
     ```bash
     npm --prefix dungeoneer-frontend run test:unit
     ```
   - Run frontend production build check:
     ```bash
     npm --prefix dungeoneer-frontend run build
     ```
3. **Spec Alignment & Task Completion:**
   - Verify OpenAPI 3.0 annotations (`@Operation`, `@ApiResponse`, `@Schema`) on all new and updated endpoints in `PlayerCharacterController` and `ReferenceDataController`.
   - Ensure all 4 delta specs in `openspec/changes/mechanical-properties-and-class-progression/specs/` accurately reflect implemented schemas, endpoints, and UI contracts.
   - Mark ALL tasks in `openspec/changes/mechanical-properties-and-class-progression/tasks.md` as completed `[x]`.
4. **Consolidated Cumulative Walkthrough:**
   - Complete `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` in Portuguese (`pt-BR`) with:
     - Consolidated summary of all changes implemented across backend and frontend.
     - Final test and build verification results.
     - Step-by-step manual validation checklist for the user (XP bar and Level Up Wizard with unrestricted multiclassing, character sheet Inventory/Feats/Actions tabs, dynamic Homebrew forms, and Encyclopedia progression table).
     - Confirmation of readiness to execute `sdd-spec-archive`.

#### Quality Gates & Verification
- Run complete test verification across workspace:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Commit locally:
  ```bash
  git add openspec/changes/mechanical-properties-and-class-progression/
  git commit -m "docs(openspec): complete walkthrough, verify security gates, and mark tasks done for mechanical-properties-and-class-progression"
  ```

#### Phase Handoff & Final Walkthrough Completion
Ensure `openspec/changes/mechanical-properties-and-class-progression/walkthrough.md` is fully consolidated. Inform the user in Portuguese (`pt-BR`) that all implementation phases, security checks, and quality gates have passed, and that the change is ready for archival via `sdd-spec-archive`.
