# Delta Spec: `player-character-domain`

> **Change ID:** `004-currency-manager-and-coin-conversion`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. `CurrencyPouch` Value Object & Conversion Rates
- Exchange rates in Copper Pieces (`CP`):
  - `1 CP (PC) = 1 CP`
  - `1 SP (PP) = 10 CP`
  - `1 EP (PE) = 50 CP`
  - `1 GP (PO) = 100 CP`
  - `1 PP (PL) = 1000 CP`
- `PUT /api/characters/{id}/currency` updates the character's 5 coin slots (`cp`, `sp`, `ep`, `gp`, `pp`).
- `POST /api/characters/{id}/items` accepts `purchaseWithCurrency: boolean`; when `true`, validates `canAfford` and deducts the item's cost using automatic coin conversion.
