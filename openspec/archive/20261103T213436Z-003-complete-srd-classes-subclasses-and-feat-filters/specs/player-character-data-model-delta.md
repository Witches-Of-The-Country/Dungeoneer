# Delta Spec: `player-character-data-model`

> **Change ID:** `003-complete-srd-classes-subclasses-and-feat-filters`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Complete SRD 2014 & 2024 Progression Seed Data
- All 13 classes (`Barbarian`, `Bard`, `Cleric`, `Druid`, `Fighter`, `Monk`, `Paladin`, `Ranger`, `Rogue`, `Sorcerer`, `Warlock`, `Wizard`, `Artificer`) MUST have Level 1–20 `archetype_feats` seeded for both `SRD_2014` and `SRD_2024`.
- Each class MUST have multiple subclasses seeded in `subclasses` along with their level-gated `subclass_feats`.

### 2. Multi-Type Feat Querying
- `GET /api/reference/feats` accepts `featTypes` (comma-separated or repeated query parameter) to filter by `ORIGIN`, `GENERAL`, `CLASS`, `SUBCLASS`, and `LINEAGE`.
