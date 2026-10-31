# Delta Spec: `frontend-web-app`

> **Change ID:** `add-encyclopedia-browse`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Encyclopedia Browse Views
- Routes:
  - `/encyclopedia/classes`: Browsing archetypes with ruleset filters.
  - `/encyclopedia/spells`: Browsing spells with circle (level 0–9), school, and ruleset filters.
  - `/encyclopedia/feats`: Browsing feats with ruleset filters.
  - `/encyclopedia/items`: Browsing equipment, weapons, armor, and magic items with category and rarity filters.
  - `/encyclopedia/lineages`: Browsing lineages/species with ruleset filters.

### 2. Client-Side Data Architecture
- TanStack Query `useInfiniteQuery` hooks in `lib/api/encyclopedia.ts`:
  - `useInfiniteSpells`, `useInfiniteFeats`, `useInfiniteItems`, `useInfiniteArchetypes`, `useInfiniteLineages`.
  - Continuous scrolling via `IntersectionObserver` sentinel component.
  - 300ms debounced full-text search triggering query refetches without page reloads.

### 3. CC-BY 4.0 Attribution
- Footer component on `/encyclopedia/*` routes declaring compliance with the Wizards of the Coast System Reference Document (SRD 5.1 & SRD 5.2) under Creative Commons Attribution 4.0 International license.
