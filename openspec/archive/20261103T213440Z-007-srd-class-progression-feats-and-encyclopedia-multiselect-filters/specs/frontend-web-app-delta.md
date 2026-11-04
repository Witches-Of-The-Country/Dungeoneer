# frontend-web-app Delta for Spec 007

## Summary of Changes
- Fix Item Rarity filter in `app/encyclopedia/items/page.tsx` by transmitting backend enum keys (`COMMON`, `UNCOMMON`, etc.).
- Fix Item Category filter in `app/encyclopedia/items/page.tsx` to align with backend item types and category representations.
- Convert single `<select>` controls into multi-select toggle chip bars with OR logic in `/encyclopedia/spells` (Levels 0-9, Schools) and `/encyclopedia/items` (Categories, Rarities).
- Update `lib/api/encyclopedia.ts` filter interfaces to accept string/number arrays.
- Upgrade Level Up Wizard Feat Selection (`components/character-sheet/level-up-wizard-modal.tsx`):
  - Replace raw HTML `<select>` with a searchable, paginated, categorized Feat card selector.
  - Client-side search and category filtering (All, Origin, General, Fighting Style, Epic Boon, Homebrew).
  - Compact pagination (5-6 items per page) with page controls and result count.
  - "Ver Detalhes" modal/pop-up showing complete prerequisite requirements and feat description.
  - Explicit selection highlight state and confirmation summary.
