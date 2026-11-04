# Tasks: Character Frontend

> **Change ID:** `character-frontend`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Install authorized dependencies (`@tanstack/react-query`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`, `sonner`) and test devDependencies (`vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`, `@playwright/test`) in `dungeoneer-frontend`
- [x] Implement API client (`lib/api/client.ts`), Auth context/hooks (`lib/api/auth.ts`), and TanStack Query hooks (`lib/api/characters.ts`)
- [x] Connect `/login`, `/register`, and `Navbar` to `auth-service` via Gateway and add route protection for `/characters/**`
- [x] Implement Character List page (`app/characters/page.tsx`) with character cards, empty state, and delete confirmation dialog
- [x] Implement 6-step Character Creation Wizard (`app/characters/new/page.tsx`) with live reference data (`SRD_2014` & `SRD_2024`), level `1–20` + HP method selector, ASI selector, and `4d6kh3` single-chance drag-and-drop (`@dnd-kit/core`) + Standard Array + Point Buy
- [x] Implement 8-tab Character Sheet page (`app/characters/[id]/page.tsx`) with live `Resumo`, `Atributos`, `Perícias`, and `Notas` tabs and "Em breve" states for `Ações`, `Magias`, `Inventário`, and `Talentos`
- [x] Implement Character Edit page (`app/characters/[id]/edit/page.tsx`) for `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes`

## Security Validation Tasks

- [x] Verify Zod validation schemas on Login, Register, Wizard, and Edit Character forms
- [x] Confirm `Authorization: Bearer <token>` is sent on `/api/characters/**` requests and `401` responses redirect to `/login`
- [x] Confirm no secrets are hardcoded in frontend code

## Shift-Left Test Suite Tasks

### Happy Path
- [x] Write Vitest + Testing Library tests for the 6-step Character Creation Wizard including `4d6kh3` single-chance drag-and-drop assignment (TP-01)
- [x] Write Vitest + Testing Library tests for the 8-tab Character Sheet view (`/characters/[id]`) (TP-02)
- [x] Write Vitest + Testing Library tests for Character List, Edit form, and Delete confirmation flow (TP-03)

### Edge Cases & Failure Modes
- [x] Write Vitest test verifying the `4d6kh3` roll button cannot be clicked a second time and wizard blocks progression if any ability slot is unassigned (TE-01)
- [x] Write Vitest test verifying RFC 7807 `ProblemDetail` errors display a `pt-BR` toast notification (TE-02)

## Documentation Tasks

- [x] Update `specs/frontend-web-app-delta.md` with the 8-tab Character Sheet layout and `4d6kh3` drag-and-drop wizard specifications

## Review Gate

- [x] All implementation tasks complete
- [x] All security validation tasks complete
- [x] All tests passing (`npm run lint && npm run build && npx vitest run`)
- [x] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [x] Ready for `sdd-spec-archive`
