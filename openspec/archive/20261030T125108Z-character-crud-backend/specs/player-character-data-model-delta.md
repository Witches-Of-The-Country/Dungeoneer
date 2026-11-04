# Delta Spec: `player-character-data-model`

> **Change ID:** `character-crud-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

---

## 1. Schema Modifications: `player_character.characters`

The `player_character.characters` table is updated with CRUD tracking, HP calculation metadata, level-up choice tracking, and strict tenant isolation:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `UUID` | `NOT NULL` | Foreign key logical placeholder to authenticated user in `auth-service`. Strictly enforced for IDOR prevention. |
| `notes` | `TEXT` | `NOT NULL DEFAULT ''` | Player free-text notes for campaign tracking. |
| `hp_calculation_method` | `VARCHAR(15)` | `NOT NULL DEFAULT 'MAX'` | Method used to compute character HP (`MAX`, `AVERAGE`, `ROLLED`). |
| `has_pending_level_choices` | `BOOLEAN` | `NOT NULL DEFAULT FALSE` | Flag set to `TRUE` when a character is created at level > 1, deferred until interactive level-up wizards are implemented. |

**Indexes**:
- `CREATE INDEX IF NOT EXISTS idx_characters_user_id ON player_character.characters(user_id);`

---

## 2. Table Creation: `player_character.class_progressions`

Represents class progression and level associations per character, supporting future multiclassing:

```sql
CREATE TABLE IF NOT EXISTS player_character.class_progressions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    character_id UUID NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE,
    archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id),
    subclass_id UUID NULL,
    class_level INT NOT NULL CHECK (class_level >= 1 AND class_level <= 20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_class_progression_character_archetype UNIQUE (character_id, archetype_id)
);

CREATE INDEX IF NOT EXISTS idx_class_progressions_character_id ON player_character.class_progressions(character_id);
```

---

## 3. Table Creation: `player_character.ability_rolls`

Persists single-chance server-side `4d6kh3` ability score rolls:

```sql
CREATE TABLE IF NOT EXISTS player_character.ability_rolls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    rolled_scores INT[] NOT NULL,
    dice_breakdown JSONB NOT NULL DEFAULT '[]',
    used BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_unused_ability_roll
    ON player_character.ability_rolls(user_id)
    WHERE used = FALSE;
```

> **Single-Chance Guarantee**: The partial unique index `idx_unused_ability_roll` guarantees that each user can hold at most one unused ability roll token at any time, preventing roll farming or browser refresh exploitation.

---

## 4. Table Creation: `player_character.hp_rolls`

Persists single-chance server-side Hit Die rolls for characters generated at level > 1 using the `ROLLED` HP method:

```sql
CREATE TABLE IF NOT EXISTS player_character.hp_rolls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id),
    target_level INT NOT NULL CHECK (target_level >= 2 AND target_level <= 20),
    rolled_hp_per_level INT[] NOT NULL,
    used BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_unused_hp_roll
    ON player_character.hp_rolls(user_id, archetype_id, target_level)
    WHERE used = FALSE;
```

> **Single-Chance Guarantee**: The partial unique index `idx_unused_hp_roll` ensures idempotency for a specific `(user_id, archetype_id, target_level)` combination until consumed upon character creation.
