# Delta Spec: `frontend-web-app`

> **Change ID:** `007-add-trinket-equipment-slot`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### Inventory Equipment Slots Grid (12 Slots)
- `InventoryTab` renders 12 equipment slots in a responsive grid (`Peito`, `Capacete`, `Luvas`, `Botas`, `Calças`, `Capa`, `Colar`, `Acessório / Cinto`, `Mão Direita`, `Mão Esquerda`, `Anel 1`, `Anel 2`).
- Slot header counter reflects `(12 Slots)` and `X / 12 ocupados`.
- Infer default slot detects belt, charm, and trinket keywords assigning them to `TRINKET`.
- Slot item removal supported via "Tirar" button on `equipment-slot-trinket`.
