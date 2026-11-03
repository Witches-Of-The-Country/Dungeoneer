Focus: Frontend / Next.js 16 (React 19) / Tailwind CSS / TanStack Query
Task: Execute Phase 3: Frontend Catalog & Store Filters Alignment

## Execution Context
- Active Plan: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/tasks.md`
- Proposal Reference: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/proposal.md`
- Scope: Execute Phase 3 only (Tasks 3.1 to 3.5). Align frontend category and rarity filters across `app/encyclopedia/items/page.tsx` and `components/character-sheet/item-catalog-modal.tsx` with canonical backend enums, run unit tests and build verification in `dungeoneer-frontend`, and append execution progress to the cumulative walkthrough. Do not proceed to subsequent phases.

## Implementation Steps
1. **Update Encyclopedia Items Page Filters (`dungeoneer-frontend/app/encyclopedia/items/page.tsx`):**
   - Align rarity filters to map cleanly to uppercase backend enums: `COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY`, `ARTIFACT` (while preserving pt-BR localized labels for UI rendering).
   - Align category tabs to support canonical categories: `ALL`, `WEAPON`, `ARMOR`, `POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT` with localized display names.
2. **Update Item Catalog & Store Modal Filters (`dungeoneer-frontend/components/character-sheet/item-catalog-modal.tsx`):**
   - Update category and rarity selectors to match the canonical backend enums and encyclopedia taxonomy (`WEAPON`, `ARMOR`, `POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`).
   - Ensure query parameters and state sent to API client/services conform to uppercase enums.
3. **Execute Verification in `dungeoneer-frontend`:**
   - Run unit tests: `npm --prefix dungeoneer-frontend run test:unit`
   - Run typecheck and production build: `npm --prefix dungeoneer-frontend run build`

## Quality Gates & Verification
- Run local validation:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Commit locally:
  ```bash
  git commit -m "feat(frontend): align encyclopedia and item catalog modal filters with backend enums"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared cumulative walkthrough artifact (`openspec/changes/008-srd-item-catalog-seed-and-category-normalization/walkthrough.md` in pt-BR) appending Phase 3 execution details:
1. List of files created/modified in this phase.
2. Verification and test results (`npm run test:unit` and `npm run build`).
3. Handoff context prepared for Phase 4 (Container Build & End-to-End Verification).
Pause and prompt the user to commit progress before proceeding to Phase 4.
