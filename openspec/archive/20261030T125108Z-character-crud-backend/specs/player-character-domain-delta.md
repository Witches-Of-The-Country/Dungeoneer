# Delta Spec: `player-character-domain`

> **Change ID:** `character-crud-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

---

## 1. REST API Endpoints & Contracts

All endpoints on `/api/characters/**` require the `X-User-Id` request header (`UUID`). If absent, the server returns an RFC 7807 `401 Unauthorized` ProblemDetail. All character and roll queries are strictly filtered by `user_id` to guarantee tenant isolation and prevent Insecure Direct Object References (IDOR / BOLA). Querying, updating, or deleting an entity belonging to another user returns an RFC 7807 `404 Not Found` ProblemDetail.

### 1.1 Single-Chance Server-Side Roll Endpoints

- **`POST /api/characters/ability-rolls`**
  - **Headers**: `X-User-Id: <UUID>`
  - **Behavior**: If the user already has an unused roll record (`used = false`), returns that record (`200 OK`) so refreshing or reloading the wizard does not grant new rolls. Otherwise, rolls 6 scores using standard `4d6kh3` (rolling 4d6, dropping lowest, summing top 3, recording the 4d6 breakdown for UI inspection), persists the roll with `used = false`, and returns `201 Created`.
  - **Response (`201 Created` / `200 OK`)**:
    ```json
    {
      "rollId": "uuid",
      "userId": "uuid",
      "rolledScores": [16, 14, 13, 12, 10, 8],
      "diceBreakdown": "[[5,4,4,1], ...]",
      "isNew": true,
      "createdAt": "2026-10-30T12:00:00Z"
    }
    ```

- **`POST /api/characters/hp-rolls`**
  - **Headers**: `X-User-Id: <UUID>`
  - **Request Body**:
    ```json
    {
      "archetypeId": "uuid",
      "level": 3
    }
    ```
  - **Validation**: `archetypeId` must exist; `level` must be between 2 and 20.
  - **Behavior**: Returns existing unused roll (`200 OK`) for `(userId, archetypeId, targetLevel)` if present. Otherwise rolls the archetype's Hit Die (1 to `hitDice.max`) for levels `2..level` (generating `level - 1` rolls), persists with `used = false`, and returns `201 Created`.
  - **Response (`201 Created` / `200 OK`)**:
    ```json
    {
      "rollId": "uuid",
      "userId": "uuid",
      "archetypeId": "uuid",
      "targetLevel": 3,
      "rolledHpPerLevel": [8, 7],
      "isNew": true,
      "createdAt": "2026-10-30T12:00:00Z"
    }
    ```

### 1.2 Character Creation (`POST /api/characters`)

- **Headers**: `X-User-Id: <UUID>`
- **Request Body (`CreateCharacterRequest`)**:
  ```json
  {
    "name": "Thorin Oakenshield",
    "level": 1,
    "ruleset": "SRD_2024",
    "lineageId": "uuid",
    "backgroundId": "uuid",
    "archetypeId": "uuid",
    "subclassId": null,
    "abilityScoreMethod": "STANDARD_ARRAY",
    "abilityScores": {
      "strength": 15,
      "dexterity": 14,
      "constitution": 13,
      "intelligence": 12,
      "wisdom": 10,
      "charisma": 8
    },
    "abilityScoreBonuses": {
      "STRENGTH": 2,
      "CONSTITUTION": 1
    },
    "abilityRollId": null,
    "hpCalculationMethod": "MAX",
    "hpRollId": null,
    "notes": "Campaign notes"
  }
  ```
- **Response (`201 Created`)**:
  - Sets `Location: /api/characters/{id}`
  - Returns `CharacterSheetResponse` containing hydrated character attributes, ability scores with applied ASI, lineage, background, archetype, class progression, saving throws, and 18 skill modifiers.

### 1.3 Character Sheet Query, Update & Deletion

- **`GET /api/characters`**
  - Returns `200 OK` with `List<CharacterSummaryResponse>` for all characters owned by `X-User-Id` ordered by `updated_at DESC`.
- **`GET /api/characters/{id}`**
  - Returns `200 OK` with full hydrated `CharacterSheetResponse`. Returns `404 Not Found` if character does not exist or belongs to another user.
- **`PUT /api/characters/{id}`**
  - **Request Body (`UpdateCharacterRequest`)**:
    ```json
    {
      "name": "Thorin II Oakenshield",
      "hitPointsCurrent": 24,
      "hitPointsTemp": 5,
      "notes": "Updated campaign notes"
    }
    ```
  - **Behavior**: Updates **only** `name`, `hitPointsCurrent` (`0 <= hp <= hitPointsMax`), `hitPointsTemp` (`>= 0`), and `notes`. Immutable fields (level, ruleset, ability scores, lineage, background, max HP) are strictly guarded against modification. Returns `200 OK` with updated `CharacterSheetResponse`, or `404 Not Found` if not owned by `X-User-Id`.
- **`DELETE /api/characters/{id}`**
  - Deletes character (cascading to ability scores and class progression) if owned by `X-User-Id`, returning `204 No Content`. Returns `404 Not Found` if not owned.

---

## 2. Server-Side Rules Validation & Mechanics

### 2.1 Ability Score Validation (`AbilityScoreValidator`)
- **`STANDARD_ARRAY`**: Base scores must be an exact permutation of `[15, 14, 13, 12, 10, 8]`.
- **`POINT_BUY`**: Each score must be within `8..15`. Total cost according to 5e point buy cost table (`8:0, 9:1, 10:2, 11:3, 12:4, 13:5, 14:7, 15:9`) must equal exactly 27 points.
- **`ROLLED`**: Requires an unused `abilityRollId` belonging to `X-User-Id`. The 6 assigned base scores must be an exact multiset permutation of the roll's `rolled_scores`. Once validated, the roll token is marked `used = true` atomically.

### 2.2 Dual Ruleset ASI Resolution (`RulesEngine`)
- **`SRD_2014` (`Srd2014RulesEngine`)**:
  - Ability Score Increases originate from `Lineage.abilityBonuses`.
  - Additional flexible bonuses (e.g. Half-Elf) are merged if specified.
  - Final scores after bonuses are capped at 20.
- **`SRD_2024` (`Srd2024RulesEngine`)**:
  - Lineages grant no ability bonuses; bonuses originate from the selected `Background`.
  - User supplies `abilityScoreBonuses` mapped to abilities listed in `Background.allowedAbilityBonuses`.
  - Permitted bonus distributions:
    - **Option A**: `+2` to one ability and `+1` to another distinct ability (`[1, 2]`).
    - **Option B**: `+1` to three distinct abilities (`[1, 1, 1]`).
  - Any selection outside the background's allowed list or with an invalid distribution throws `CharacterValidationException` (`422 Unprocessable Entity`).
  - Final scores after bonuses are capped at 20.

### 2.3 Server-Side Hit Points Calculation
- **Level 1**: Always receives `hitDice.max + CON_mod` (minimum 1 HP).
- **Levels 2..level**:
  - **`MAX`**: Each level adds `hitDice.max + CON_mod` (minimum 1 per level).
  - **`AVERAGE`**: Each level adds `(hitDice.max / 2 + 1) + CON_mod` (minimum 1 per level).
  - **`ROLLED`**: Requires `hpRollId` belonging to `X-User-Id`, matching `archetypeId` and `targetLevel == level`. Sums `rolled_hp_per_level[i] + CON_mod` (minimum 1 per level) and marks the roll `used = true` atomically.
- Characters created at level > 1 have `hasPendingLevelChoices = true`.

### 2.4 Derived Attributes
- **Armour Class**: Base 10 + DEX modifier.
- **Initiative Modifier**: DEX modifier.
- **Proficiency Bonus**: `(level - 1) / 4 + 2`.
- **Speed**: Inherited from `Lineage.speed` (default 30).
- **Saving Throws**: Base ability modifier + proficiency bonus if the ability is included in `Archetype.savingThrowProficiencies`.
- **Skills (18 Skills)**: Base ability modifier + proficiency bonus if the skill is included in `Background.skillProficiencies`.
