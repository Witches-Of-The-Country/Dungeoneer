# Delta Spec: `player-character-domain`

> **Change ID:** `add-long-short-rest`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates
- Short rest: spend chosen Hit Dice (server-rolled die + CON mod, min 0, capped at max HP); recover short-rest feat charges.
- Long rest: full HP, temp HP kept, recover half Hit Dice (min 1), all spell slots, long-rest charges, exhaustion −1 (min 0).
- Rest logic delegated to `RulesEngine` (`SRD_2014`/`SRD_2024`).
- Invariant: characters at 0 HP/dead cannot rest.
