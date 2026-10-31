# Delta Spec: `player-character-data-model`

> **Change ID:** `add-encyclopedia-browse`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### 1. New Reference Tables (Schema `player_character`)

#### Table: `player_character.spells`
```sql
CREATE TABLE player_character.spells (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ruleset VARCHAR(20) NOT NULL CHECK (ruleset IN ('SRD_2014', 'SRD_2024')),
    slug VARCHAR(150) NOT NULL,
    name_en VARCHAR(150) NOT NULL,
    name VARCHAR(150) NOT NULL,
    level INT NOT NULL,
    school VARCHAR(50) NOT NULL,
    casting_time VARCHAR(50),
    range_area VARCHAR(100),
    components VARCHAR(50),
    duration VARCHAR(100),
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_spells_ruleset_slug UNIQUE (ruleset, slug)
);
```

#### Table: `player_character.feats`
```sql
CREATE TABLE player_character.feats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ruleset VARCHAR(20) NOT NULL CHECK (ruleset IN ('SRD_2014', 'SRD_2024')),
    slug VARCHAR(150) NOT NULL,
    name_en VARCHAR(150) NOT NULL,
    name VARCHAR(150) NOT NULL,
    prerequisites VARCHAR(255),
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_feats_ruleset_slug UNIQUE (ruleset, slug)
);
```

#### Table: `player_character.items`
```sql
CREATE TABLE player_character.items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ruleset VARCHAR(20) NOT NULL CHECK (ruleset IN ('SRD_2014', 'SRD_2024')),
    slug VARCHAR(150) NOT NULL,
    name_en VARCHAR(150) NOT NULL,
    name VARCHAR(150) NOT NULL,
    item_category VARCHAR(30) NOT NULL, -- EQUIPMENT, WEAPON, ARMOR, MAGIC
    item_type VARCHAR(50) NOT NULL,
    rarity VARCHAR(30) NOT NULL,        -- COMMON, UNCOMMON, RARE, VERY_RARE, LEGENDARY, ARTIFACT, NONE
    attunement BOOLEAN NOT NULL DEFAULT FALSE,
    cost VARCHAR(50),
    weight VARCHAR(30),
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_items_ruleset_slug UNIQUE (ruleset, slug)
);
```

### 2. Full-Text Search (GIN) & Indexing Strategy
- PostgreSQL GIN indexes using `to_tsvector('portuguese', coalesce(name, '') || ' ' || coalesce(name_en, '') || ' ' || coalesce(description, ''))` on `spells`, `feats`, and `items`.
- Compound B-Tree indexes for alphabetical cursor pagination: `(name ASC, id ASC)`.
- Filtering B-Tree indexes: `(ruleset, level)` on `spells`, `(ruleset, item_category, rarity)` on `items`.

### 3. Flyway Seed Migrations (V8, V9, V10)
- `V8__seed_srd_spells.sql`: SRD 5.1 and 5.2 spells in Portuguese with original English names.
- `V9__seed_srd_feats.sql`: SRD 5.1 and 5.2 feats in Portuguese with original English names.
- `V10__seed_srd_items.sql`: SRD 5.1 and 5.2 equipment, weapons, armor, and magic items.
- All INSERT statements adhere to idempotency constraint `ON CONFLICT (ruleset, slug) DO NOTHING`.
