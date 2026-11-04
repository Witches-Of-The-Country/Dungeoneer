# Delta Spec: `player-character-data-model`

> **Change ID:** `mechanical-properties-and-class-progression`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Flyway Migration `V16__add_mechanical_properties_and_class_progression_tables.sql`

#### 1.1 `player_character.items` — Mechanical Properties JSONB
```sql
ALTER TABLE player_character.items
    ADD COLUMN IF NOT EXISTS mechanical_properties JSONB NOT NULL DEFAULT '{}'::jsonb;

CREATE INDEX IF NOT EXISTS idx_items_mechanical_properties
    ON player_character.items USING GIN (mechanical_properties);
```

- **Polymorphic JSONB Structure (`mechanical_properties`)**:
  - **Weapon (`type = "WEAPON"`)**:
    ```json
    {
      "type": "WEAPON",
      "damageRolls": [
        { "diceFormula": "1d8", "damageType": "SLASHING" },
        { "diceFormula": "1d6", "damageType": "FIRE" }
      ],
      "versatileDice": "1d10",
      "magicBonus": 1,
      "properties": ["VERSATILE", "FINESSE", "LIGHT", "HEAVY", "REACH", "TWO_HANDED", "THROWN", "AMMUNITION", "LOADING"]
    }
    ```
  - **Armor / Robe / Shield (`type = "ARMOR"`)**:
    ```json
    {
      "type": "ARMOR",
      "armorCategory": "LIGHT | MEDIUM | HEAVY | ROBE | SHIELD",
      "baseAc": 14,
      "magicBonus": 0,
      "strengthRequirement": 0,
      "stealthDisadvantage": false
    }
    ```
  - **Consumable (`type = "CONSUMABLE"`)**:
    ```json
    {
      "type": "CONSUMABLE",
      "effectType": "HEAL_HP | GRANT_TEMP_HP | DEAL_DAMAGE | BUFF_UTILITY",
      "diceFormula": "2d4+2",
      "effectDescription": "Recupera 2d4 + 2 pontos de vida imediatamente."
    }
    ```

#### 1.2 `player_character.feats` — Feat Type Discriminator & Mechanical Effects JSONB
```sql
ALTER TABLE player_character.feats
    ADD COLUMN IF NOT EXISTS feat_type VARCHAR(30) NOT NULL DEFAULT 'GENERAL'
        CHECK (feat_type IN ('GENERAL', 'ORIGIN', 'CLASS', 'SUBCLASS', 'LINEAGE')),
    ADD COLUMN IF NOT EXISTS mechanical_effects JSONB NOT NULL DEFAULT '{}'::jsonb;

CREATE INDEX IF NOT EXISTS idx_feats_feat_type ON player_character.feats (feat_type);
```

- **JSONB Structure (`mechanical_effects`)**:
  ```json
  {
    "abilityBonuses": { "STR": 1 },
    "acBonus": 0,
    "initiativeBonus": 0,
    "speedBonus": 0,
    "hpPerLevelBonus": 2,
    "grantedProficiencies": ["Heavy Armor"],
    "actionName": "Retomar o Fôlego (Second Wind)",
    "actionType": "BONUS_ACTION",
    "diceFormula": "1d10",
    "usesPerRest": 1,
    "restType": "SHORT"
  }
  ```

#### 1.3 `player_character.archetypes` & `player_character.subclasses`
```sql
ALTER TABLE player_character.archetypes
    ADD COLUMN IF NOT EXISTS subclass_level INT NOT NULL DEFAULT 3 CHECK (subclass_level BETWEEN 1 AND 20);

ALTER TABLE player_character.subclasses
    ADD COLUMN IF NOT EXISTS slug VARCHAR(120),
    ADD COLUMN IF NOT EXISTS name_en VARCHAR(120),
    ADD COLUMN IF NOT EXISTS description TEXT NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS source VARCHAR(50) NOT NULL DEFAULT 'SRD',
    ADD COLUMN IF NOT EXISTS owner_id UUID NULL;

CREATE UNIQUE INDEX IF NOT EXISTS uq_subclasses_ruleset_slug
    ON player_character.subclasses (ruleset, slug);
```

#### 1.4 Associative Class & Subclass Feature Progression Tables (`archetype_feats` & `subclass_feats`)
```sql
CREATE TABLE IF NOT EXISTS player_character.archetype_feats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id) ON DELETE CASCADE,
    feat_id UUID NOT NULL REFERENCES player_character.feats(id) ON DELETE CASCADE,
    unlock_level INT NOT NULL CHECK (unlock_level BETWEEN 1 AND 20),
    UNIQUE (archetype_id, feat_id, unlock_level)
);

CREATE TABLE IF NOT EXISTS player_character.subclass_feats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subclass_id UUID NOT NULL REFERENCES player_character.subclasses(id) ON DELETE CASCADE,
    feat_id UUID NOT NULL REFERENCES player_character.feats(id) ON DELETE CASCADE,
    unlock_level INT NOT NULL CHECK (unlock_level BETWEEN 1 AND 20),
    UNIQUE (subclass_id, feat_id, unlock_level)
);
```

#### 1.5 `player_character.characters`, `character_items` & `character_feats`
```sql
ALTER TABLE player_character.characters
    ADD COLUMN IF NOT EXISTS experience_points INT NOT NULL DEFAULT 0 CHECK (experience_points >= 0),
    ADD COLUMN IF NOT EXISTS campaign_id UUID NULL;

ALTER TABLE player_character.character_feats
    ADD COLUMN IF NOT EXISTS id UUID DEFAULT gen_random_uuid(),
    ADD COLUMN IF NOT EXISTS unlocked_at_level INT NOT NULL DEFAULT 1,
    ADD COLUMN IF NOT EXISTS current_charges INT NULL,
    ADD COLUMN IF NOT EXISTS max_charges INT NULL,
    ADD COLUMN IF NOT EXISTS choices JSONB NOT NULL DEFAULT '{}'::jsonb;
```

### 2. Flyway Migrations `V17` & `V18` — SRD 2014 & 2024 Compendium Seed & Mechanical Extraction
- **`V17__seed_srd_subclasses_and_progression_feats.sql`**:
  - Seeds all 13 SRD Subclasses across both `SRD_2014` and `SRD_2024`:
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
  - Seeds `CLASS` and `SUBCLASS` feats for levels 1–20 (e.g., *Rage*, *Unarmored Defense*, *Reckless Attack*, *Extra Attack*, *Bardic Inspiration*, *Channel Divinity*, *Wild Shape*, *Second Wind*, *Action Surge*, *Indomitable*, *Flurry of Blows*, *Divine Smite*, *Lay on Hands*, *Sneak Attack*, *Cunning Action*, *Uncanny Dodge*, *Evasion*, *Font of Magic*, *Metamagic*, *Eldritch Invocations*, *Arcane Recovery*) and links them in `archetype_feats` and `subclass_feats`.
- **`V18__populate_srd_mechanical_properties.sql`**:
  - Extracts and populates `mechanical_properties` on all SRD weapons, armors, shields, robes, and healing potions in `player_character.items`.
  - Updates `feat_type` (`ORIGIN`, `GENERAL`) and `mechanical_effects` on all SRD feats in `player_character.feats` (including *Tough* `hpPerLevelBonus: 2`, *Alert* `initiativeBonus`, and ability score bonuses on half-feats/Epic Boons).
