# Delta Spec: `player-character-domain`

> **Change ID:** `add-homebrew-feat-charges`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates
- Max charges: `FIXED` | `ABILITY_MODIFIER` (author-chosen ability, min 1) | `PROFICIENCY` | `LEVEL`; recalculated dynamically.
- Recovery per rest type (short/long independent): `NONE` | `ALL` | `FIXED X` | `MODIFIER` | `PROFICIENCY` | `LEVEL`; long default `ALL`; always clamped to max.
- Legacy `usesPerRest`/`restType` mapped at read time.
