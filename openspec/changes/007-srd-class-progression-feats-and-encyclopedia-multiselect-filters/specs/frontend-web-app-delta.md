# frontend-web-app Delta for Spec 007

## Summary of Changes
- Fix Item Rarity filter in `app/encyclopedia/items/page.tsx` by transmitting backend enum keys (`COMMON`, `UNCOMMON`, etc.).
- Fix Item Category filter in `app/encyclopedia/items/page.tsx` to align with backend item types and category representations.
- Convert single `<select>` controls into multi-select toggle chip bars with OR logic in `/encyclopedia/spells` (Levels 0-9, Schools) and `/encyclopedia/items` (Categories, Rarities).
- Update `lib/api/encyclopedia.ts` filter interfaces to accept string/number arrays.
