Focus: Full-Stack / Spring Boot (Java 21 / PostgreSQL / Clean Architecture) & Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Create SDD Change Proposal for Mechanical Properties, Class Progression & Character Sheet Evolution

## Objective
Structure a formal change proposal following the project's Spec-Driven Development conventions.

## Instructions
1. Activate skill `sdd-spec-create` under `openspec/changes/`.
2. Target change name: `mechanical-properties-and-class-progression`
3. Required proposal elements:
   - Problem & Scope:
     - Mechanical Item Persistence & Polymorphic Domain Model:
       * Common base item attributes preserved in the `items` table, enhanced with typed `mechanical_properties JSONB` mapped to pure domain Value Objects (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`) without framework annotations in the domain layer.
       * Weapons (`WeaponMechanics`): composite damage rolls (`damageRolls`: e.g., `1d8 PIERCING` + `1d6 FIRE`), optional versatile die (`versatileDice`: e.g., `1d10`), magic bonus (`magicBonus`), and weapon properties (`FINESSE`, `LIGHT`, `HEAVY`, `REACH`, `TWO_HANDED`, `THROWN`, `AMMUNITION`, `LOADING`, `VERSATILE`).
       * Armor (`ArmorMechanics`): base AC (`baseAc`), magic bonus (`magicBonus`), strength requirement (`strengthRequirement`), stealth disadvantage (`stealthDisadvantage`), and armor classification categories (`armorCategory`):
         - `LIGHT`: Base AC + full DEX modifier.
         - `MEDIUM`: Base AC + DEX modifier (capped at +2).
         - `HEAVY`: Fixed base AC (no DEX modifier applied).
         - `ROBE`: Base AC + full DEX modifier (qualifies as unarmored for class mechanics such as Monk/Wizard and permits additive AC bonuses).
         - `SHIELD`: Flat additive AC bonus (e.g., +2).
       * Consumables (`ConsumableMechanics`): structured effect classification (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`), optional dice formulas (e.g., `2d4+2`), and effect descriptions. Using an item from the character sheet executes a server-side roll, applies HP or temporary HP modifications directly to the character, and decrements `quantity` by 1 (removing the item from inventory when zero).
     - Unified Feats & Class/Subclass/Lineage Feature Architecture:
       * Unified storage under the `feats` table with `feat_type` discriminator (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`) and `mechanical_effects JSONB` mapped to domain Value Objects:
         - Ability score bonuses / half-feat ASI choices.
         - Passive bonuses (`acBonus`, `initiativeBonus`, `speedBonus`, `hpPerLevelBonus` as in the *Tough* feat).
         - Granted proficiencies (skills, armor, weapons, saving throws).
         - Active actions and rolls (e.g., Rage, Sneak Attack, Second Wind) with resource charge tracking (`usesPerRest`, `restType: SHORT | LONG`).
       * Associative progression tables: `archetype_feats (archetype_id, feat_id, unlock_level)` and `subclass_feats (subclass_id, feat_id, unlock_level)` mapping feature progression from levels 1 to 20.
     - Complete SRD (2014 & 2024) Data Migration, Seed & Homebrew Support:
       * Flyway database migration extracting and populating `mechanical_properties` on existing SRD items and `mechanical_effects` on existing SRD feats in the database.
       * Comprehensive seed data for all 13 SRD subclasses (1 per class across `SRD_2014` and `SRD_2024`) and level 1 to 20 Class & Subclass Feats for all SRD classes.
       * Dynamic Homebrew creation dialog (`CreateHomebrewDialog`) supporting Weapons, Armor/Robes/Shields, Consumables, Feats with mechanical effects, Classes (with level 1–20 feature assignments), and Subclasses.
       * Class Encyclopedia (`/encyclopedia/classes`) updated to render complete level 1–20 progression tables and subclass capabilities.
     - XP Tracker, Level Up & Multiclassing Engine (UC2):
       * Character persistence attributes: `experience_points INT NOT NULL DEFAULT 0` and `campaign_id UUID NULLABLE` (reserved placeholder for campaign locks where only the DM may grant XP).
       * Official D&D 5e XP progression bar (0 to 355,000 XP across levels 1–20) in character sheet header with interactive `+ Add XP` modal (disabled when `campaign_id != null`).
       * Level-up trigger indicator (pulsing CTA) upon meeting XP threshold, alongside Milestone-based level-up that automatically advances the character's level and sets XP to the floor of that new level.
       * Multi-step Level Up Wizard:
         - Class selection to advance existing class level or introduce an unrestricted multiclass (no ability score >= 13 prerequisite restrictions).
         - Subclass selection prompt when reaching the archetype's `subclass_level`.
         - HP calculation choice: fixed average, maximum, or single server-validated die roll (persisted in `hp_rolls`) + CON modifier + per-level feat bonuses (e.g., *Tough*).
         - Automatic unlocking of level-specific Class and Subclass Feats.
         - Ability Score Improvement (ASI) branch at class ASI thresholds (4, 8, 12, 16, 19, plus Fighter 6 & 14, Rogue 10): choose either Ability Score Increase (+2 or +1/+1, max 20) OR select a Feat from the catalog.
     - Full Activation of Character Sheet Dashboard Tabs (`/characters/[id]`):
       * Replace "Coming Soon" placeholders with fully functional interactive panels:
         - **Inventory Tab**: Item management (add, remove, quantity adjustments, carrying capacity / weight calculation), equip/unequip toggles with dynamic AC recalculation according to armor categories, attunement tracker (max 3 slots), and one-click 'Use' action for consumables.
         - **Feats Tab**: Categorized view of Origin/General Feats, Lineage Traits, Class Features, and Subclass Features with active charge tracking per short/long rest.
         - **Actions Tab**: Auto-generated attack entries from equipped weapons (to-hit bonus calculation, damage dice including versatile options), plus active class/feat abilities with interactive dice rollers.
   - Impacted Areas (invoke `sdd-code-explorer` if needed)
   - Delta Specs under `openspec/changes/mechanical-properties-and-class-progression/specs/` targeting canonical specs:
     - `player-character-data-model-delta.md`: Flyway migration scripts, schema alterations for `items`, `feats`, `characters`, new associative progression tables (`archetype_feats`, `subclass_feats`), JSONB schema definitions, and seed data.
     - `player-character-domain-delta.md`: Clean Architecture domain value objects (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, `MechanicalEffects`), rules engine strategies for AC computation, damage formula resolution, Level Up & Multiclass use cases, and rest recharge mechanisms.
     - `frontend-web-app-delta.md`: UI specifications for XP progress bar, Level Up Wizard, updated Homebrew modal with dynamic mechanical forms, Class Encyclopedia progression table, and character sheet tabs (Inventory, Feats, Actions).
     - `infrastructure-and-testing-delta.md`: Testcontainers integration test harnesses and shift-left unit test suites.
   - Atomic Task Breakdown (`tasks.md`):
     - Sequenced, incremental execution phases covering:
       * Phase 1: Database schema migrations, JSONB mappings, and complete SRD seed data.
       * Phase 2: Domain value objects, rules engines (AC calculation, attack/damage resolution), and use case ports.
       * Phase 3: Application services, DTOs with Jakarta Validation, REST endpoints, and integration tests.
       * Phase 4: Level Up & Multiclassing wizard and XP progression backend/frontend integration.
       * Phase 5: Character sheet tabs activation (Inventory, Feats, Actions) and interactive roll mechanics.
       * Phase 6: Homebrew creation dialog enhancements and Class Encyclopedia progression views.
       * Phase 7: Verification, end-to-end integration tests, and JaCoCo coverage validation (>= 80%).

Do NOT modify production source code. Present proposal artifact for user review.
