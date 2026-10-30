# Delta Spec: `player-character-data-model`

> **Change ID:** `character-crud-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

1. **`characters` Table Updates**:
   - `user_id UUID NOT NULL` (logical reference to authenticated user in `auth-service`).
   - `notes TEXT NOT NULL DEFAULT ''`.
   - `hp_calculation_method VARCHAR(15) NOT NULL DEFAULT 'MAX'` (`MAX`, `AVERAGE`, `ROLLED`).
   - `has_pending_level_choices BOOLEAN NOT NULL DEFAULT FALSE`.
2. **`ability_rolls` Table**:
   - `id UUID PK`, `user_id UUID NOT NULL`, `rolled_scores INT[] NOT NULL`, `dice_breakdown JSONB NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
   - Partial unique index on `(user_id) WHERE used = FALSE` to guarantee single-chance roll per user until consumed.
3. **`hp_rolls` Table**:
   - `id UUID PK`, `user_id UUID NOT NULL`, `archetype_id UUID NOT NULL FK -> archetypes.id`, `target_level INT NOT NULL CHECK (2-20)`, `rolled_hp_per_level INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
