# Change Proposal: 004 — Character & Item Currency Manager with Automatic Coin Conversion

> **Change ID:** `004-currency-manager-and-coin-conversion`
> **Status:** approved
> **Created:** 2026-11-01

---

## Summary

Implement a full D&D 5e Currency Manager (`PC/CP`, `PP/SP`, `PE/EP`, `PO/GP`, `PL/PP`) across `player-character-service` and `dungeoneer-frontend`:
1. Separate coin slots (`cp`, `sp`, `ep`, `gp`, `pp`) persisted on `PlayerCharacter` and structured coin cost (`costCoins` / parsed `cost`) on `Item` (`CreateHomebrewDialog` + Item catalog).
2. Automatic D&D 5e conversion engine (`1 PL = 10 PO = 20 PE = 100 PP = 1000 PC`, i.e., in copper units: `CP=1, SP=10, EP=50, GP=100, PP=1000`) supporting manual coin updates, consolidating/exchanging coins, and purchasing items from the catalog (`Comprar com Moedas`) with automatic change calculation when the character has sufficient total wealth.

---

## Motivation & Problem Statement

Currently, `Item.cost` is an unstructured string (`"50 po"`) and `PlayerCharacter` has zero currency fields. Players and DMs need to track each coin denomination in its own slot (`PC`, `PP`, `PE`, `PO`, `PL`), convert between denominations, define structured multi-coin item prices in Homebrew creation, and purchase items in `InventoryTab` (and future Campaign shops) with automatic coin conversion/deduction.

---

## Scope

### In Scope
- Adding `copper_pieces`, `silver_pieces`, `electrum_pieces`, `gold_pieces`, and `platinum_pieces` columns to `player_character.characters` via Flyway migration `V21`.
- Creating pure-domain Value Object `CurrencyPouch` in `domain/model/CurrencyPouch.java` with `long` copper conversion math (`CP=1, SP=10, EP=50, GP=100, PP=1000`), `canAfford()`, `deductWithConversion()`, `consolidate()`, and `parseCostString()`.
- Exposing `PUT /api/characters/{id}/currency` and adding `purchaseWithCurrency` support to `POST /api/characters/{id}/items`.
- Rendering the 5-slot Currency Manager card, coin conversion calculator, and `"Comprar"` action in `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx`.
- Adding 5 separate coin slot inputs (`PC`, `PP`, `PE`, `PO`, `PL`) in `CreateHomebrewDialog` (Item tab).

### Out of Scope
- Multiplayer merchant NPC haggling or campaign-shared party treasury (scheduled for Campaign Service Phase 3).

---

## Proposed Solution

1. **Database & Domain (`player-character-service`)**:
   - Flyway migration `V21__add_character_currency_columns.sql` adding `copper_pieces INT NOT NULL DEFAULT 0`, `silver_pieces INT NOT NULL DEFAULT 0`, `electrum_pieces INT NOT NULL DEFAULT 0`, `gold_pieces INT NOT NULL DEFAULT 0`, and `platinum_pieces INT NOT NULL DEFAULT 0` to `characters`.
   - Domain Value Object `CurrencyPouch` (`int cp, int sp, int ep, int gp, int pp`) in `domain/model/CurrencyPouch.java` (zero framework imports) with methods:
     - `long toTotalCopper()`: `cp + sp*10L + ep*50L + gp*100L + pp*1000L`.
     - `boolean canAfford(CurrencyPouch cost)`: `this.toTotalCopper() >= cost.toTotalCopper()`.
     - `CurrencyPouch deductWithConversion(CurrencyPouch cost)`: deducts exact coins where possible and automatically breaks higher-denomination coins into change when needed.
     - `CurrencyPouch consolidate()`: exchanges lower denominations into higher denominations (`PO`/`PL`).
     - `static CurrencyPouch parseCostString(String cost)`: parses `"15 gp"`, `"50 po"`, `"5 pp, 2 pc"`, etc.
   - REST Endpoint `PUT /api/characters/{id}/currency` to update coins or convert denominations, and optional `purchaseWithCurrency: boolean` flag on `POST /api/characters/{id}/items` to deduct item cost upon purchase.
2. **Frontend Currency Manager (`dungeoneer-frontend`)**:
   - Structured 5-slot coin cost inputs (`PC`, `PP`, `PE`, `PO`, `PL`) in `CreateHomebrewDialog` (Item tab) that format into `cost` and `mechanicalProperties.costCoins`.
   - Interactive `CurrencyPouchCard` in `InventoryTab` displaying the 5 coin slots (`PC`, `PP`, `PE`, `PO`, `PL`), quick `+ / -` adjustment, `"Converter Moedas"` button, total wealth in Gold (`PO`), and a `"Comprar"` button alongside `"Adicionar"` when adding items from the catalog.

---

## Impacted Files

| File Path | Layer | Reason |
|---|---|---|
| `player-character-service/.../db/migration/V21__add_character_currency_columns.sql` | Database | Add `copper_pieces`, `silver_pieces`, `electrum_pieces`, `gold_pieces`, `platinum_pieces` to `characters`. |
| `player-character-service/.../domain/model/CurrencyPouch.java` | Domain | Pure Java Value Object for 5-coin pouch and conversion/deduction math. |
| `player-character-service/.../domain/model/PlayerCharacter.java` | Domain | Hold `CurrencyPouch` and expose `updateCurrency()` and `spendCurrency()`. |
| `player-character-service/.../adapter/in/web/controller/PlayerCharacterController.java` | Web | Expose `PUT /api/characters/{id}/currency` and support `purchaseWithCurrency` in `addItem`. |
| `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` | Frontend UI | Render 5-slot Currency Manager, conversion calculator, and `"Comprar"` item button. |
| `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` | Frontend UI | Structured 5-slot coin inputs for Homebrew items. |

---

## Acceptance Criteria

- **Given** a character with `0 PC, 0 PP, 0 PE, 1 PO, 0 PL` (`100 PC` total value), **When** the player purchases an item costing `5 PP` (`50 PC` value) with `purchaseWithCurrency = true`, **Then** `CurrencyPouch.deductWithConversion` deducts `1 PO` and returns `5 PP` in change (`0 PO, 5 PP`), updating the character's persisted currency pouch.
- **Given** a DM creating a Homebrew Item, **When** they enter coin values in the 5 coin slots (`PC`, `PP`, `PE`, `PO`, `PL`), **Then** the structured coin cost is saved and displayed on the item card and inventory purchase modal.

---

## Defensive Security & Validation Considerations

- **Non-Negative Coin Invariant (`@Min(0)` / `@Max(1_000_000_000)`)**: `UpdateCurrencyRequest` and `CurrencyPouch` constructor enforce `cp >= 0, sp >= 0, ep >= 0, gp >= 0, pp >= 0` to prevent negative coin balance exploits.
- **64-bit Overflow Prevention**: `CurrencyPouch.toTotalCopper()` uses `long` 64-bit arithmetic (`10L`, `50L`, `100L`, `1000L`) so large coin hoards never overflow 32-bit `int`.
- **Ownership Verification**: `PUT /api/characters/{id}/currency` and `POST /api/characters/{id}/items` verify character ownership against `X-Auth-User-Id` inside a `@Transactional` service boundary.

---

## Testing Strategy & Shift-Left Plan

| Scenario | Type | Target Test Class / File | Expected Outcome |
|---|---|---|---|
| Exact and cross-denomination coin deduction (`1 PO` - `5 PP` = `5 PP`) | Unit | `CurrencyPouchTest.java` | Returns `CurrencyPouch(0, 5, 0, 0, 0)` (`50 CP` remaining). |
| Purchase with insufficient funds (`4 PP` vs `1 PO` item) | Unit | `CurrencyPouchTest.java`, `InventoryServiceTest.java` | Throws `IllegalArgumentException` / `DomainValidationException` (HTTP 400/422) and leaves coins unchanged. |
| Negative coin input rejection | Unit | `CurrencyPouchTest.java` | Throws `IllegalArgumentException` when any coin slot `< 0`. |
| Parse legacy SRD cost strings (`"50 po"`, `"15 gp"`, `"2 pp, 5 pc"`) | Unit | `CurrencyPouchTest.java` | Accurately parses into `CurrencyPouch` denominations. |
| Frontend 5-slot currency manager & purchase button | Unit | `dungeoneer-frontend/__tests__/character-sheet.test.tsx` | Displays `PC, PP, PE, PO, PL` slots and triggers currency update/purchase mutations. |

- **Verification Commands**:
  - `mvn -f player-character-service/backend/pom.xml test`
  - `npm --prefix dungeoneer-frontend run test:unit`

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Legacy SRD items with free-text `cost` (e.g. `"10 po"`, `"25 gp"`, `"—"`) | Low | `CurrencyPouch.parseCostString()` supports both pt-BR (`pc, pp, pe, po, pl`) and English (`cp, sp, ep, gp, pp`) abbreviations and defaults unparseable/free items to `0`. |
| Breaking higher coins into copper when partial change is needed | Low | `deductWithConversion` greedily preserves existing untouched denominations first and only converts the minimum necessary higher coins into change from largest to smallest denomination (`PP -> GP -> SP -> CP`). |
