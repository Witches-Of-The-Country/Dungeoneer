# Proposal: Character Frontend

> **Change ID:** `character-frontend`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Implement the full end-to-end Player Character experience in `dungeoneer-frontend` (Next.js 16, React 19, TypeScript 5, TailwindCSS 4) connected to `api-gateway` (`:9090`):
1. Integrate `/login` and `/register` with `POST /api/auth/login` and `POST /api/auth/register`, persisting the JWT and protecting `/characters/**` routes.
2. Build the Character List page (`/characters`) displaying the logged-in user's characters as Dark Fantasy RPG cards (`name`, `level`, `class`, `lineage`, `HP`, `ruleset`) ordered by `updated_at DESC`.
3. Migrate the 6-step prototype wizard (`/prototype/character-creation`) into the production Character Creation Wizard (`/characters/new`), loading SRD 2014/2024 classes, lineages, and backgrounds from `/api/reference/*`, adding initial level (`1–20`) and HP calculation method (`MAX`, `AVERAGE`, `ROLLED` single-chance), ruleset ASI selection (`2014` lineage ASI vs `2024` background `+2/+1` or `+1/+1/+1`), and the 3 ability score methods (`STANDARD_ARRAY`, `POINT_BUY` 27-point counter, and `ROLLED` `4d6kh3` single-chance roll with `@dnd-kit/core` drag-and-drop assignment across the 6 attributes).
4. Build the Character Sheet page (`/characters/[id]`) using an **8-tab layout** (`Resumo`, `Atributos`, `Perícias`, `Ações`, `Magias`, `Inventário`, `Talentos`, `Notas`), where `Resumo`, `Atributos`, `Perícias`, and `Notas` render live interactive data (including inline HP adjustments and auto-saved/editable `Notas`) and `Ações`, `Magias`, `Inventário`, and `Talentos` render a themed "Em breve" empty state.
5. Build the Character Edit page (`/characters/[id]/edit`) for editing `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes`, plus physical character deletion (`DELETE /api/characters/{id}`) with a confirmation dialog.

## Motivation

`dungeoneer-frontend` currently has static `/login` and `/register` forms and an unintegrated prototype at `/prototype/character-creation`, with no `/characters` routes, no API client, and no TanStack Query setup. Completing this change delivers the complete vertical slice of Phase 1 Character Sheet Management in Portuguese (`pt-BR`).

## Scope

### In Scope
- **Dependencies (`package.json`)**:
  - Install `@tanstack/react-query`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`, `sonner` (shadcn toast), and devDependencies `vitest`, `@vitejs/plugin-react`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`, `@playwright/test`.
- **Auth & API Client Layer (`lib/api/`)**:
  - Fetch wrapper targeting `process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:9090'`, automatically attaching `Authorization: Bearer <token>`, parsing RFC 7807 `ProblemDetail` errors into `pt-BR` toast notifications, and redirecting to `/login` on `401 Unauthorized`.
  - Auth context/hooks (`useAuth`, `login`, `register`, `logout`) connected to `/login`, `/register`, and `Navbar`.
  - TanStack Query provider and hooks (`useCharacters`, `useCharacter(id)`, `useCreateCharacter`, `useUpdateCharacter`, `useDeleteCharacter`, `useRollAbilityScores`, `useRollHitPoints`, `useReferenceClasses`, `useReferenceLineages`, `useReferenceBackgrounds`).
- **Character List (`/characters`)**:
  - Responsive card grid showing each character's name, ruleset badge (`5e 2014` / `2024`), level & class, lineage, HP bar, and quick actions (`Abrir Ficha`, `Editar`, `Excluir` with confirmation modal), plus a prominent `Criar Personagem` CTA button linking to `/characters/new`.
- **6-Step Character Creation Wizard (`/characters/new`)**:
  - Follows the exact sequence of `/prototype/character-creation`:
    1. **Passo 1 — Nome & Regras**: Character name (`1–100` chars), Ruleset selector (`SRD_2014` vs `SRD_2024`), Initial Level (`1–20`), and (when level > 1) HP Method (`Média`, `Máximo`, `Rolado` with single-chance button calling `POST /api/characters/hp-rolls`).
    2. **Passo 2 — Linhagem**: Fetches `GET /api/reference/lineages?ruleset=...`, displays traits, speed, size, and (for `SRD_2014`) fixed ASI bonuses.
    3. **Passo 3 — Classe**: Fetches `GET /api/reference/classes?ruleset=...`, displays hit dice, primary ability, saving throws, and spellcasting indicator.
    4. **Passo 4 — Antecedente**: Fetches `GET /api/reference/backgrounds?ruleset=...`, displays skill proficiencies, and (for `SRD_2024`) informational Origin Feat card + interactive ASI selector (`+2/+1` or `+1/+1/+1` among the 3 allowed abilities).
    5. **Passo 5 — Atributos**: 3 tabs:
       - `Rolagem (4d6kh3)`: Single-chance "Rolar 4d6kh3" button calling `POST /api/characters/ability-rolls`. Displays the 6 rolled value chips (locked to 1 roll per creation; refreshing reloads the same server roll) and lets the user drag-and-drop (`@dnd-kit/core`) or swap the 6 numbers onto the 6 ability slots (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`), showing base + ASI = final score and modifier.
       - `Valores Padrão`: Drag-and-drop / swap of `[15, 14, 13, 12, 10, 8]` across the 6 ability slots.
       - `Compra de Pontos`: `+/-` controls (`8..15`) with live 27-point budget counter.
    6. **Passo 6 — Revisão**: Summary of all choices, calculated HP, AC, Initiative, Proficiency Bonus, and "Criar Personagem" submit button (`POST /api/characters` → redirects to `/characters/[id]`).
- **8-Tab Character Sheet (`/characters/[id]`)**:
  - Header with character name, level, class, lineage, background, ruleset badge, `hasPendingLevelChoices` badge (if created > level 1), `Editar` button, and `Excluir` button with confirmation dialog.
  - Tab navigation with 8 tabs:
    1. `Resumo`: Core stats (HP current/max/temp with quick adjust, AC, Initiative, Speed, Proficiency Bonus, Hit Dice, Saving Throws).
    2. `Atributos`: 6 ability cards with final score, modifier, and saving throw proficiency indicator.
    3. `Perícias`: All 18 D&D skills in `pt-BR`, showing governing ability, proficiency dot (from background), and calculated total modifier.
    4. `Notas`: Editable textarea for character notes/backstory saved via `PUT /api/characters/{id}`.
    5. `Ações`: Themed "Em breve" empty state.
    6. `Magias`: Themed "Em breve" empty state.
    7. `Inventário`: Themed "Em breve" empty state.
    8. `Talentos`: Themed "Em breve" empty state (plus informational display of 2024 Origin Feat if applicable).
- **Character Edit (`/characters/[id]/edit`)**:
  - Form (`react-hook-form` + `zod`) allowing editing of `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes`, submitting `PUT /api/characters/{id}`.

### Out of Scope
- Level-up wizard (`/characters/[id]/level-up`) and Spell Search page (`/spells`).

## Proposed Solution

### Approach
We will build a modular `lib/api/` client and wrap `app/layout.tsx` with `QueryClientProvider`, `AuthProvider`, and `Toaster`. The wizard in `app/characters/new/page.tsx` will reuse the visual language and `Stepper` component from `app/prototype/character-creation/page.tsx`, replacing hardcoded mock arrays with TanStack Query hooks and adding `@dnd-kit/core` droppable/draggable slots for assigning rolled (`4d6kh3`) and standard array scores across the 6 attributes.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/package.json` | Modify | Add `@tanstack/react-query`, `@dnd-kit/core`, `sonner`, Vitest, Testing Library, Playwright |
| `dungeoneer-frontend/app/layout.tsx` | Modify | Wrap with `QueryProvider`, `AuthProvider`, and `Toaster` |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Show `Meus Personagens` link and logged-in user / logout button |
| `dungeoneer-frontend/components/login-form.tsx` | Modify | Connect to `POST /api/auth/login` and redirect to `/characters` |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Connect to `POST /api/auth/register` and redirect to `/characters` |
| `dungeoneer-frontend/lib/api/client.ts` | Create | Fetch wrapper with Bearer token & RFC 7807 `ProblemDetail` parsing |
| `dungeoneer-frontend/lib/api/auth.ts` | Create | Auth API functions & state management |
| `dungeoneer-frontend/lib/api/characters.ts` | Create | Character CRUD, ability-rolls, hp-rolls, and reference data hooks |
| `dungeoneer-frontend/app/characters/page.tsx` | Create | Character list grid with delete confirmation modal |
| `dungeoneer-frontend/app/characters/new/page.tsx` | Create | 6-step Character Creation Wizard with `4d6kh3` drag-and-drop |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Create | 8-tab Character Sheet view |
| `dungeoneer-frontend/app/characters/[id]/edit/page.tsx` | Create | Character edit form (`name`, `HP`, `notes`) |

### Data Model Changes (if applicable)
Frontend TypeScript interfaces (`CharacterResponse`, `CreateCharacterPayload`, `AbilityRollResponse`, `ReferenceClass`, `ReferenceLineage`, `ReferenceBackground`) aligned 1:1 with backend DTOs.

## Acceptance Criteria

- [ ] **Given** an unauthenticated visitor **When** accessing `/characters` or `/characters/new` **Then** they are redirected to `/login`; after logging in or registering, they are redirected to `/characters`.
- [ ] **Given** the 6-step wizard at `/characters/new` **When** the user selects `Rolagem (4d6kh3)` on Step 5 and clicks "Rolar Atributos" **Then** `POST /api/characters/ability-rolls` is called once, the roll button is locked so no second roll is possible (even after page refresh), and the user can drag-and-drop or swap the 6 rolled values across `FOR`, `DES`, `CON`, `INT`, `SAB`, and `CAR`.
- [ ] **Given** a completed 6-step wizard **When** the user clicks "Criar Personagem" on Step 6 **Then** `POST /api/characters` succeeds (`201 Created`) and the browser navigates to `/characters/[id]`.
- [ ] **Given** the Character Sheet at `/characters/[id]` **When** rendered **Then** all 8 tabs (`Resumo`, `Atributos`, `Perícias`, `Ações`, `Magias`, `Inventário`, `Talentos`, `Notas`) are navigable, with live data in `Resumo`, `Atributos`, `Perícias`, and `Notas`, and "Em breve" states in `Ações`, `Magias`, `Inventário`, and `Talentos`.
- [ ] **Given** `/characters/[id]/edit` **When** the user updates `name`, `hitPointsCurrent`, `hitPointsTemp`, or `notes` **Then** `PUT /api/characters/{id}` persists the changes and updates the sheet; clicking `Excluir` and confirming in the dialog calls `DELETE /api/characters/{id}` and returns to `/characters`.

## Defensive Security & Validation Considerations

### Input Validation
- Client-side forms validated with Zod + React Hook Form (`name` 1–100 chars, Point Buy 27 points, all 6 ability scores assigned, `0 <= hitPointsCurrent <= hitPointsMax`).

### Secret Isolation
- Only `NEXT_PUBLIC_API_URL` is used on the client; no secrets or private keys exist in frontend code.

### Authorization & Authentication Impact
- Protected routes check auth state and attach `Authorization: Bearer <token>` on all API calls through `api-gateway`.

### Other Security Risks
- React escapes all user-entered text (`name`, `notes`) by default, preventing XSS.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | 6-step wizard renders steps, handles Point Buy / Standard Array / single-chance `4d6kh3` drag-and-drop assignment, and submits | Component (Vitest + Testing Library) | Calls `POST /api/characters` with expected payload |
| TP-02 | Character sheet (`/characters/[id]`) renders all 8 tabs and switches between `Resumo`, `Atributos`, `Perícias`, `Notas`, and "Em breve" tabs | Component (Vitest + Testing Library) | Displays accurate modifiers, 18 skills, and notes |
| TP-03 | Edit form updates `name`, `HP`, and `notes`; Delete modal confirms and calls `DELETE` | Component (Vitest) & E2E (Playwright) | Updates/deletes character and navigates properly |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Attempt to click "Rolar 4d6kh3" a second time or advance without assigning all 6 ability slots | Component (Vitest) | Roll button is disabled after single roll; Next button blocked until all 6 slots assigned |
| TE-02 | Backend returns `422` or `409` `ProblemDetail` | Component (Vitest) | Displays error message toast in `pt-BR` |

### Test Commands
- **Run unit tests:** `npx vitest run`
- **Run E2E tests:** `npx playwright test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Drag-and-drop usability on mobile/touch or keyboard | Medium | Medium | Configure `@dnd-kit/core` with `PointerSensor` + `KeyboardSensor` and also allow tap-to-select / tap-to-swap fallback on slots |

## Specification Changes & Delta Stubs

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Update `/characters/[id]` from an 8-card grid to an **8-tab layout** (`Resumo`, `Atributos`, `Perícias`, `Ações`, `Magias`, `Inventário`, `Talentos`, `Notas`), document the single-chance `4d6kh3` drag-and-drop (`@dnd-kit/core`) interaction in Step 5 of the Wizard, and document initial level (`1–20`) + HP method selection on Step 1.

## References

- `openspec/specs/frontend-web-app.md`
- `dungeoneer-frontend/app/prototype/character-creation/page.tsx`
