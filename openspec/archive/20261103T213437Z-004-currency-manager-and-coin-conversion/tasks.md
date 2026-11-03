# Tasks: 004 — Character & Item Currency Manager with Automatic Coin Conversion

> **Change ID:** `004-currency-manager-and-coin-conversion`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Create Flyway migration `V21__add_character_currency_columns.sql` adding `copper_pieces`, `silver_pieces`, `electrum_pieces`, `gold_pieces`, and `platinum_pieces` to `characters`.
- [x] Create pure domain Value Object `CurrencyPouch.java` with D&D 5e conversion math (`CP=1, SP=10, EP=50, GP=100, PP=1000`), `canAfford`, `deductWithConversion`, and cost string parsing.
- [x] Update `PlayerCharacter.java`, `PlayerCharacterJpaEntity.java`, `CharacterSheetHydrator.java`, `InventoryService.java`, and `PlayerCharacterController.java` (`PUT /api/characters/{id}/currency` and `purchaseWithCurrency` in `POST /api/characters/{id}/items`).
- [x] Update `dungeoneer-frontend/lib/api/characters.ts`, `components/character-sheet/inventory-tab.tsx`, and `components/homebrew/create-homebrew-dialog.tsx` with the 5-slot Currency Manager (`PC`, `PP`, `PE`, `PO`, `PL`), conversion tool, structured item coin cost inputs, and `"Comprar com Moedas"` action.

## Test Tasks

- [x] Add unit tests for `CurrencyPouch` (exact deduction, cross-denomination change calculation, insufficient funds error, and cost string parsing) and `InventoryServiceTest`.
- [x] Add frontend Vitest tests verifying the 5-slot currency manager and conversion calculations.

## Documentation Tasks

- [x] Document the Currency Pouch model and endpoints in `specs/player-character-domain-delta.md` and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
