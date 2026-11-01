# Delta Spec: `frontend-web-app`

> **Change ID:** `add-homebrew-creation`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Role-Gated UI Elements

- In encyclopedia / compendium pages (`/compendium/spells`, `/compendium/items`, etc.), a **"Criar novo"** action button is displayed.
- The button is rendered conditionally based on the user's authenticated session:
  - If user is logged in with `DM` role (`roles.includes('DM')`): button is visible.
  - If user is player or anonymous: button is hidden.

### 2. Creation Dialogs & Form Validation

- A reusable modal component `CreateHomebrewDialog` is added in `components/homebrew/create-homebrew-dialog.tsx`.
- Form controls use React Hook Form with Zod schemas matching backend validation rules:
  - Field labels, placeholders, and error messages are localized in Portuguese (`pt-BR`).
  - Inputs sanitize markdown and prevent malicious HTML injection.

### 3. API Client & Cache Management

- Defined in `lib/api/homebrew.ts`:
  - `useCreateHomebrewSpell()`
  - `useCreateHomebrewItem()`
  - `useCreateHomebrewFeat()`
  - `useCreateHomebrewLineage()`
  - `useCreateHomebrewArchetype()`
- Each mutation hook invalidates the associated query key (e.g. `['reference', 'spells']`) upon success, triggering an automatic refresh of the data table.
- A toast notification in `pt-BR` (e.g., "Magia Homebrew criada com sucesso!") is displayed upon completion.
