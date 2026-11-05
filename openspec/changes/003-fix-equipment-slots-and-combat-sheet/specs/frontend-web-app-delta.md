# Delta Spec: `frontend-web-app`

> **Change ID:** `003-fix-equipment-slots-and-combat-sheet`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### Inventory Equipment Slots Grid, Temp HP Damage & Custom Action CRUD
- `InventoryTab` renders the 11 equipment slots (`Peito`, `Capacete`, `Luvas`, `Botas`, `Calças`, `Capa`, `Colar`, `Mão Direita`, `Mão Esquerda`, `Anel 1`, `Anel 2`) and assigns/replaces slots when equipping items.
- Character Sheet HP controls apply negative deltas (`handleApplyDamage`, `-1 PV`) to `hitPointsTemp` first before reducing `hitPointsCurrent`.
- `ActionsTab` provides Edit and Delete controls on all cards with `category === 'CUSTOM'`.
