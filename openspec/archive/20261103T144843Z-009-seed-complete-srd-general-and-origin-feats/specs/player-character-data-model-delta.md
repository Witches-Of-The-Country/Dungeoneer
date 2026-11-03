# Delta Spec: Player Character Data Model — SRD General & Origin Feats

> **Change ID:** `009-seed-complete-srd-general-and-origin-feats`
> **Target Document:** `openspec/specs/player-character-data-model.md`

---

## 1. Flyway Migration `V26__seed_complete_srd_general_and_origin_feats.sql`

A new Flyway migration script `V26` is added to seed the complete catalog of General and Origin Feats:

### Table Affected: `player_character.feats`
- Deterministic UUIDs:
  - `30000000-0000-0000-0000-000000000700` to `0799` for SRD 2014 General Feats.
  - `40000000-0000-0000-0000-000000000700` to `0799` for SRD 2024 General Feats.
  - `40000000-0000-0000-0000-000000000800` to `0810` for SRD 2024 Origin Feats.
- Categorization Updates:
  - `UPDATE player_character.feats SET feat_type = 'ORIGIN' WHERE slug IN ('magic-initiate-2024', 'savage-attacker-2024', 'skilled-2024') AND ruleset = 'SRD_2024';`
- Payload Schema for `mechanical_effects`:
```json
{
  "abilityBonuses": { "STR": 1 },
  "acBonus": 0,
  "initiativeBonus": 0,
  "speedBonus": 0,
  "hpPerLevelBonus": 0,
  "grantedProficiencies": [],
  "abilityBonusChoice": {
    "chooseCount": 1,
    "allowedAbilities": ["STR", "DEX", "CON"],
    "bonusAmount": 1
  },
  "prerequisiteRules": [
    {
      "type": "ABILITY_SCORE",
      "targetKey": "STRENGTH",
      "minValue": 13,
      "displayLabel": "Força 13+"
    }
  ],
  "spellChoice": {
    "cantripsCount": 2,
    "spellsCount": 1,
    "allowedClasses": ["CLERIC", "DRUID", "WIZARD"],
    "spellLevel": 1
  }
}
```
