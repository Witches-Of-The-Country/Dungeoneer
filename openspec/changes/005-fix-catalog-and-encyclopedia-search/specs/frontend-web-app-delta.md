# Delta Spec: `frontend-web-app`

> **Change ID:** `005-fix-catalog-and-encyclopedia-search`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### Item Catalog & Encyclopedia Search UX
- `ItemCatalogModal` normalizes diacritics (`String.prototype.normalize("NFD")`) for instant client-side filtering and debounces backend `q` requests without emptying results on prefix keystrokes.
