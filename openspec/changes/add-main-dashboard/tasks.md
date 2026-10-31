# Tasks: Add Main Dashboard

> **Change ID:** `add-main-dashboard`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [ ] Extend `useAuth` hook with `isDM` and `isPlayer` boolean properties
- [ ] Verify/ensure login and register redirects point to `/characters`
- [ ] Create `EncyclopediaMenu` component with Dark Fantasy styling and Lucide icons
- [ ] Implement DM-only gating and "Mestre" badge on Monsters and NPCs cards
- [ ] Create DM-only placeholder page for Monsters (`/encyclopedia/monsters`) with "Em breve" empty state and route guard
- [ ] Create DM-only placeholder page for NPCs (`/encyclopedia/npcs`) with "Em breve" empty state and route guard
- [ ] Integrate `EncyclopediaMenu` into main dashboard layout (`app/characters/page.tsx`)
- [ ] Update `Navbar` with encyclopedia navigation link and DM badge indicator

## Test Tasks

- [ ] Write unit tests for `useAuth` role flags (`isDM`, `isPlayer`) covering all role combinations and unauthenticated states
- [ ] Write unit tests for `EncyclopediaMenu` verifying category rendering and DM visibility gating
- [ ] Write unit tests for Monsters and NPCs placeholder pages verifying DM access and non-DM restriction
- [ ] Write integration tests for the unified dashboard page verifying character list and encyclopedia sections
- [ ] Execute complete frontend unit test suite and production build

## Documentation Tasks

- [ ] Complete `proposal.md`, `specs/frontend-web-app-delta.md`, and `tasks.md`
- [ ] Maintain cumulative `walkthrough.md` in pt-BR documenting all phases and test results

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
