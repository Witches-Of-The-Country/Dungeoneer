# Tasks: Remove Unwanted Currency Auto-Consolidation on Item Purchase

> **Change ID:** `006-fix-shop-currency-auto-conversion`
> **Execution Order:** `006`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Refactor `CurrencyPouch.deductWithConversion(long costInCopper)` in `player-character-service` so it deducts the item cost without calling `fromCopper(remaining)` on the entire wallet, preserving non-participating coin denominations (`CP`, `SP`, `EP`, `GP`, `PP`).

## Test Tasks

- [x] Update `CurrencyPouchTest.java` and `InventoryServiceTest.java` to verify that purchasing an item preserves unaffected coin denominations (including `EP` and un-consolidated `CP`/`SP`) while still making change when breaking a higher coin.

## Documentation Tasks

- [x] Update `specs/player-character-domain-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`mvn clean test`)
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
