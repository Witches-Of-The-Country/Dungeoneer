# frontend-web-app Delta for Spec 006

## Summary of Changes
- New detail modals: `SpellDetailModal`, `FeatDetailModal`, `ItemDetailModal`, and `LineageDetailModal`.
- Wire `onClick` on `ReferenceCard` across all encyclopedia views (`/encyclopedia/{spells,feats,items,lineages}`).
- Add "Editar" button inside detail modals when entry has `ruleset === 'HOMEBREW'`.
- Fix currency input layout in `components/homebrew/create-homebrew-dialog.tsx` so numbers up to multiple digits (e.g., 50 po) are clearly legible.
- In `components/character-sheet/feats-tab.tsx`, show real feat name, source badge (Lineage, Archetype, Background, Custom), and allow clicking card to open `FeatDetailModal`.
- In `components/character-sheet/inventory-tab.tsx`, replace the `<select>` item picker with `ItemCatalogModal`.
- `ItemCatalogModal` provides search, filtering, item detail inspection, Direct Addition (free/loot), and Store Checkout (deducting coins automatically with currency conversion).
