# Delta Spec: `player-character-data-model`

> **Change ID:** `add-homebrew-feat-charges`  
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`  
> **Change Type:** Modify

---

## 1. Schema & Persistence Updates

### 1.1 JSONB Schema in `feats.mechanical_effects`
- No relational DDL table alterations required.
- The `feats.mechanical_effects` JSONB document gains the following optional fields:
  ```json
  {
    "maxChargesType": "FIXED | ABILITY_MODIFIER | PROFICIENCY | LEVEL",
    "chargeAbility": "STRENGTH | DEXTERITY | CONSTITUTION | INTELLIGENCE | WISDOM | CHARISMA",
    "fixedMaxCharges": 1,
    "shortRestRecoveryType": "NONE | ALL | FIXED | MODIFIER | PROFICIENCY | LEVEL",
    "shortRestRecoveryFixed": 1,
    "longRestRecoveryType": "ALL | NONE | FIXED | MODIFIER | PROFICIENCY | LEVEL",
    "longRestRecoveryFixed": 1
  }
  ```
- **Backward Compatibility**:
  - Legacy fields (`maxUses`, `rechargeOn`, `usesPerRest`, `restType`) remain supported and are gracefully mapped at read-time if new charge fields are missing.
  - Primitive fields in record models are boxed (`Integer`, `Boolean`) to prevent Jackson HTTP deserializer null-mapping failures.

### 1.2 DTOs & Hydration
- `CharacterFeatDto` outputs `mechanicalEffects` containing the charge configuration, along with dynamically calculated `maxCharges` and clamped `currentCharges`.
- `CharacterSheetHydrator` recomputes `maxCharges` and clamps `currentCharges` during character sheet hydration, ensuring stats updates immediately reflect on feat capacities.
