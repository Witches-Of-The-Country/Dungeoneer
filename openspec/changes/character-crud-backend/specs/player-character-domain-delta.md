# Delta Spec: `player-character-domain`

> **Change ID:** `character-crud-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

1. **Single-Chance Server-Side Roll Endpoints**:
   - `POST /api/characters/ability-rolls`: Generates 6 `4d6kh3` scores persisted in `ability_rolls` for the authenticated user (`X-User-Id`). Idempotent while an unused roll exists (`used = false`), ensuring only one chance per character creation.
   - `POST /api/characters/hp-rolls`: Generates hit die rolls for levels `2..N` persisted in `hp_rolls` for `X-User-Id` when creating a character at level > 1 with `ROLLED` HP method.
2. **Server-Side Creation & ASI Resolution (`POST /api/characters`)**:
   - Validates `STANDARD_ARRAY` (`[15, 14, 13, 12, 10, 8]`), `POINT_BUY` (27-point budget, scores `8..15`), and `ROLLED` (consumes `abilityRollId` and verifies exact multiset permutation).
   - Applies `SRD_2014` fixed lineage ASI or `SRD_2024` background ASI (`+2/+1` or `+1/+1/+1` restricted to the background's 3 allowed abilities).
   - Supports initial `level` (`1..20`) with `hpCalculationMethod` (`MAX`, `AVERAGE`, `ROLLED`) and marks `hasPendingLevelChoices = (level > 1)`.
3. **Scoped Character Sheet CRUD (`X-User-Id`)**:
   - `GET /api/characters`: Lists characters owned by `X-User-Id` ordered by `updated_at DESC`.
   - `GET /api/characters/{id}`: Returns full hydrated character sheet (including lineage, background, class progressions, 18 calculated skill modifiers, saving throws, and `notes`) if owned by `X-User-Id`.
   - `PUT /api/characters/{id}`: Updates only `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes`.
   - `DELETE /api/characters/{id}`: Physically deletes the character owned by `X-User-Id` (`204 No Content`).
