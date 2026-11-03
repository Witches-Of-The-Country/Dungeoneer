# Delta Spec: `frontend-web-app`

> **Change ID:** `004-currency-manager-and-coin-conversion`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Character & Item Currency Manager UI
- `InventoryTab` renders a 5-slot coin pouch (`PC`, `PP`, `PE`, `PO`, `PL`), conversion button, and `"Comprar"` option on catalog items.
- `CreateHomebrewDialog` (Item tab) provides 5 separate coin inputs (`PC`, `PP`, `PE`, `PO`, `PL`) to define item prices.
