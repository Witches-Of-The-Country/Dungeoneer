# Delta Spec: `player-character-domain`

> **Change ID:** `006-fix-shop-currency-auto-conversion`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Non-Consolidating Currency Deduction (`CurrencyPouch.deductWithConversion`)
- Purchasing an item deducts the item's cost from the character's `CurrencyPouch` without auto-consolidating unrelated coin denominations into Platinum or Gold, preserving existing `CP`, `SP`, `EP`, `GP`, and `PP` stacks and only breaking higher coins when change is required.
- Single-denomination exact match prioritizes `GP -> SP -> CP -> EP -> PP` without touching unrelated coins.
- When change is required, the single smallest sufficient higher coin is broken, returning standard change (`GP`, `SP`, `CP`) while leaving untouched denominations (such as `EP`) preserved.
- Full wallet consolidation remains available via the explicit `CurrencyPouch.consolidate()` method for user-initiated consolidation actions.
