# Tasks: Add Main Dashboard

> **Change ID:** `add-main-dashboard`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Extend `useAuth` hook with `isDM` and `isPlayer` boolean properties
- [x] Verify/ensure login and register redirects point to `/characters`
- [x] Create `EncyclopediaMenu` component with Dark Fantasy styling and Lucide icons
- [x] Implement DM-only gating and "Mestre" badge on Monsters and NPCs cards
- [x] Create DM-only placeholder page for Monsters (`/encyclopedia/monsters`) with "Em breve" empty state and route guard
- [x] Create DM-only placeholder page for NPCs (`/encyclopedia/npcs`) with "Em breve" empty state and route guard
- [x] Integrate `EncyclopediaMenu` into main dashboard layout (`app/characters/page.tsx`)
- [x] Update `Navbar` with encyclopedia navigation link and DM badge indicator

## Test Tasks

- [x] Write unit tests for `useAuth` role flags (`isDM`, `isPlayer`) covering all role combinations and unauthenticated states
- [x] Write unit tests for `EncyclopediaMenu` verifying category rendering and DM visibility gating
- [x] Write unit tests for Monsters and NPCs placeholder pages verifying DM access and non-DM restriction
- [x] Write integration tests for the unified dashboard page verifying character list and encyclopedia sections
- [x] Execute complete frontend unit test suite and production build

## Documentation Tasks

- [x] Complete `proposal.md`, `specs/frontend-web-app-delta.md`, and `tasks.md`
- [x] Maintain cumulative `walkthrough.md` in pt-BR documenting all phases and test results

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
