Focus: Frontend / Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query / Lucide Icons) & Client-Side Auth Role Gating
Task: Execute Full Phased Implementation of Change `add-main-dashboard`

## Execution Context
- Active Plan: `openspec/changes/add-main-dashboard/tasks.md`
- Linked Proposal: `openspec/changes/add-main-dashboard/proposal.md`
- Delta Spec: `openspec/changes/add-main-dashboard/specs/frontend-web-app-delta.md`
- Authentication Context: User authentication already exists in the application with roles `PLAYER` and `DUNGEON_MASTER`. (Note: The mention of "no auth" in `AGENTS.md` is outdated and must be disregarded).
- Core Objectives:
  1. Complete the placeholder sections of `proposal.md` (acceptance criteria in Given-When-Then format, defensive security considerations, and shift-left testing strategy) and populate `tasks.md` with atomic tasks. Await user confirmation before writing implementation code.
  2. Enhance the client-side authentication utilities (`use-auth.ts`) to expose helper methods/flags for role checking (`isDM`, `isPlayer`) based on `DUNGEON_MASTER` and `PLAYER` roles.
  3. Create the `EncyclopediaMenu` component displaying navigation cards for core reference categories: Archetypes, Spells, Feats, Items, Lineages, Monsters, and NPCs.
  4. Enforce role gating such that Monsters and NPCs are accessible only to users with the `DUNGEON_MASTER` role, rendering a Dark Fantasy "Em breve" empty state placeholder.
  5. Integrate the `EncyclopediaMenu` into the main post-login dashboard (`app/characters/page.tsx`), update the navigation bar (`navbar.tsx`), and verify login/register redirects.
- Architectural & Quality Constraints:
  - SDD Spec-First Gate: Never mutate production code while the proposal and tasks remain placeholders. Phase 1 must finalize the spec and obtain user approval.
  - Testing Strategy (TDD): Write comprehensive unit tests for auth role helpers, menu category gating, and empty state rendering before and alongside implementation.
  - Language Separation: Source code, component names, hook definitions, automated test suites, and Git commit messages strictly in English. User-facing frontend strings, category labels, empty states, tooltips, and the shared walkthrough artifact strictly in Portuguese (`pt-BR`).
  - Build & Tools: Use global `npm` commands (e.g., `npm --prefix dungeoneer-frontend run test:unit`, `npm --prefix dungeoneer-frontend run build`).
  - Phase Pauses & Conventional Commits: Execute one atomic phase at a time. After verifying quality gates, update the cumulative walkthrough (`openspec/changes/add-main-dashboard/walkthrough.md` in pt-BR), provide the exact ready-to-copy `git commit` command, and pause for user approval before advancing.

---

### Phase 1: Spec Completion & User Approval Gate

#### Execution Context
- Scope: Execute Phase 1 only. Finalize the SDD proposal, delta specifications, and implementation tasks. Do NOT modify any application source code during this phase.

#### Implementation Steps
1. Inspect `openspec/changes/add-main-dashboard/proposal.md` and complete all placeholder sections:
   - Acceptance Criteria: Formulate Given-When-Then verifiable scenarios covering:
     * Post-login landing on dashboard showing character list and encyclopedia menu.
     * Rendering of player categories (Archetypes, Spells, Feats, Items, Lineages) for both `PLAYER` and `DUNGEON_MASTER`.
     * Conditional visibility or access restriction of DM categories (Monsters, NPCs) strictly for `DUNGEON_MASTER`.
     * Empty state display ("Em breve") when navigating to Monsters and NPCs placeholders.
   - Defensive Security & Authorization Considerations:
     * Input validation: Client-side routing guards and fallback validation.
     * Secret isolation: Confirmation that no credentials or tokens are exposed.
     * Authorization & Authentication Impact: Role check enforcement (`DUNGEON_MASTER` vs `PLAYER`) for DM-only views; route guard fallback if a `PLAYER` directly visits `/encyclopedia/monsters` or `/encyclopedia/npcs`.
   - Testing Strategy & Shift-Left Plan: Detail happy path and edge-case unit test matrices (`main-dashboard.test.tsx`, `encyclopedia-menu.test.tsx`).
2. Complete `openspec/changes/add-main-dashboard/specs/frontend-web-app-delta.md`:
   - Document new UI components, routes (`/encyclopedia/monsters`, `/encyclopedia/npcs`), and navigation layout updates.
3. Populate `openspec/changes/add-main-dashboard/tasks.md`:
   - Replace placeholder items with atomic, checkable tasks categorized into Implementation, Test, and Documentation sections.
4. Present the completed proposal and task breakdown to the user and request explicit confirmation before writing code.

#### Quality Gates & Verification
- Verify markdown formatting and completeness of all sections in `openspec/changes/add-main-dashboard/`.
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/add-main-dashboard/proposal.md \
          openspec/changes/add-main-dashboard/tasks.md \
          openspec/changes/add-main-dashboard/specs/frontend-web-app-delta.md
  git commit -m "docs(spec): complete proposal and tasks for add-main-dashboard"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Initialize `openspec/changes/add-main-dashboard/walkthrough.md` in Portuguese (`pt-BR`):
  1. Summary of specification details and Given-When-Then criteria defined.
  2. Await user approval on the spec before advancing to Phase 2.

---

### Phase 2: Auth Role Extension & Route Redirect Verification

#### Execution Context
- Scope: Execute Phase 2 only. Enhance client-side auth helpers in `dungeoneer-frontend/lib/hooks/use-auth.ts` to expose `isDM` and `isPlayer` flags, and verify post-login / post-registration redirects.

#### Implementation Steps
1. Update `dungeoneer-frontend/lib/hooks/use-auth.ts`:
   - Inspect the current auth state structure (e.g., JWT decoded payload or user object containing roles).
   - Export boolean flags:
     * `isDM`: returns `true` if user roles include `DUNGEON_MASTER`.
     * `isPlayer`: returns `true` if user roles include `PLAYER`.
   - Ensure graceful fallback to `false` when user is unauthenticated or roles array is null/empty.
2. Verify redirect paths in `dungeoneer-frontend/components/login-form.tsx` and `dungeoneer-frontend/components/register-form.tsx`:
   - Confirm successful authentication redirects to the primary dashboard route (`/characters`).
3. Write unit tests in `dungeoneer-frontend/__tests__/use-auth-roles.test.ts` (or equivalent test location):
   - Test `isDM` returning true for `DUNGEON_MASTER` role.
   - Test `isDM` returning false for `PLAYER` role or unauthenticated state.
   - Test `isPlayer` returning true for `PLAYER` role.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/lib/hooks/use-auth.ts \
          dungeoneer-frontend/components/login-form.tsx \
          dungeoneer-frontend/components/register-form.tsx \
          dungeoneer-frontend/__tests__/
  git commit -m "feat(auth): expose isDM and isPlayer role flags and verify dashboard redirects"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 2 section to `openspec/changes/add-main-dashboard/walkthrough.md` in Portuguese (`pt-BR`):
  1. Description of auth role helper changes.
  2. Test execution results.
  3. Pause and present the commit command to the user before proceeding to Phase 3.

---

### Phase 3: Encyclopedia Menu Component & Category Role Gating

#### Execution Context
- Scope: Execute Phase 3 only. Build the `EncyclopediaMenu` component rendering category cards and gating DM-only items.

#### Implementation Steps
1. Create `dungeoneer-frontend/components/encyclopedia-menu.tsx`:
   - Define categories array:
     * Arquétipos (Classes): `/encyclopedia/classes`
     * Magias: `/encyclopedia/spells`
     * Talentos: `/encyclopedia/feats`
     * Itens: `/encyclopedia/items`
     * Linhagens: `/encyclopedia/lineages`
     * Monstros (DM only): `/encyclopedia/monsters`
     * NPCs (DM only): `/encyclopedia/npcs`
   - Use Lucide icons appropriate for each category (e.g., `Shield`, `Sparkles`, `Award`, `Package`, `Users`, `Skull`, `UserCheck`).
   - Use Dark Fantasy RPG UI tokens (dark slate/zinc borders, parchment gold hover accents, muted text, smooth transitions).
   - Leverage `useAuth()` to evaluate `isDM`:
     * If `isDM` is `false`, filter out or visibly lock/hide the "Monstros" and "NPCs" cards.
     * If `isDM` is `true`, render "Monstros" and "NPCs" cards with a distinct "Mestre" badge.
2. Write unit tests in `dungeoneer-frontend/__tests__/encyclopedia-menu.test.tsx`:
   - Verify all 5 player categories render for any authenticated user.
   - Verify "Monstros" and "NPCs" cards do not render when `isDM` is false.
   - Verify "Monstros" and "NPCs" render with DM badge when `isDM` is true.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/components/encyclopedia-menu.tsx \
          dungeoneer-frontend/__tests__/encyclopedia-menu.test.tsx
  git commit -m "feat(frontend): create EncyclopediaMenu component with DM role gating"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 3 section to `openspec/changes/add-main-dashboard/walkthrough.md` in Portuguese (`pt-BR`):
  1. Components created and styling decisions.
  2. Unit test results.
  3. Pause and present the commit command to the user before proceeding to Phase 4.

---

### Phase 4: Monsters and NPCs Empty State Placeholders ("Em breve")

#### Execution Context
- Scope: Execute Phase 4 only. Build the placeholder routes for Monsters and NPCs with "Em breve" empty states and client-side DM route protection.

#### Implementation Steps
1. Create `dungeoneer-frontend/app/encyclopedia/monsters/page.tsx`:
   - Route guard: check `isDM`. If `!isDM`, display access denied message ("Acesso restrito ao Mestre do Jogo") with a redirect button back to `/characters`.
   - Empty state UI:
     * Header: "Enciclopédia de Monstros"
     * Dark fantasy empty state card: Skull icon, title "Em breve", descriptive message: "O bestiário completo de criaturas e monstros do SRD está em desenvolvimento e estará disponível para Mestres em breve."
     * Action button: "Voltar ao Dashboard" navigating to `/characters`.
2. Create `dungeoneer-frontend/app/encyclopedia/npcs/page.tsx`:
   - Follow the same pattern:
     * Route guard: check `isDM`.
     * Empty state UI: "Enciclopédia de NPCs", title "Em breve", descriptive message: "A ferramenta de consulta e gestão de NPCs para campanhas estará disponível em breve."
     * Action button: "Voltar ao Dashboard".
3. Write unit tests in `dungeoneer-frontend/__tests__/encyclopedia-placeholders.test.tsx`:
   - Test that visiting `/encyclopedia/monsters` as DM shows the "Em breve" bestiary empty state.
   - Test that visiting `/encyclopedia/monsters` as PLAYER displays the restricted access message.
   - Test similar behavior for `/encyclopedia/npcs`.

#### Quality Gates & Verification
- Run frontend unit tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/encyclopedia/monsters/page.tsx \
          dungeoneer-frontend/app/encyclopedia/npcs/page.tsx \
          dungeoneer-frontend/__tests__/encyclopedia-placeholders.test.tsx
  git commit -m "feat(frontend): add Monsters and NPCs DM placeholder pages with 'Em breve' empty state"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append Phase 4 section to `openspec/changes/add-main-dashboard/walkthrough.md` in Portuguese (`pt-BR`):
  1. Detail new routes and route guard behavior.
  2. Test results.
  3. Pause and present the commit command to the user before proceeding to Phase 5.

---

### Phase 5: Dashboard Layout Integration, Navigation Bar & Final Verification

#### Execution Context
- Scope: Execute Phase 5 only. Integrate the `EncyclopediaMenu` into the main dashboard page, update navigation bar links, execute full test suites, and finalize documentation.

#### Implementation Steps
1. Modify `dungeoneer-frontend/app/characters/page.tsx`:
   - Restructure layout to present a unified post-login dashboard:
     * Section 1: "Meus Personagens" (existing character cards list with "Criar Personagem" CTA).
     * Section 2: "Enciclopédia & Referências" rendering `<EncyclopediaMenu />`.
   - Maintain responsive grid layout (1 column on mobile, 2 to 4 columns on desktop).
2. Modify `dungeoneer-frontend/components/navbar.tsx`:
   - Add direct navigation link to "Enciclopédia" or quick navigation menu.
   - If user is DM, display a small badge or indicator (e.g., "Mestre").
3. Create comprehensive dashboard integration test `dungeoneer-frontend/__tests__/main-dashboard.test.tsx`:
   - Test rendering of both character list section and encyclopedia section on `/characters`.
   - Test responsiveness and navigation interactions.
4. Execute full quality gates across frontend:
   - Run unit tests and production build.
5. Update `openspec/changes/add-main-dashboard/tasks.md`:
   - Mark all completed tasks as `[x]`.

#### Quality Gates & Verification
- Run complete test suite and production build:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/characters/page.tsx \
          dungeoneer-frontend/components/navbar.tsx \
          dungeoneer-frontend/__tests__/main-dashboard.test.tsx \
          openspec/changes/add-main-dashboard/tasks.md
  git commit -m "feat(frontend): integrate EncyclopediaMenu into main dashboard and update navbar"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Complete the shared walkthrough `openspec/changes/add-main-dashboard/walkthrough.md` in Portuguese (`pt-BR`):
  1. Consolidated summary of all changes made across phases (auth role helpers, menu component, placeholder pages, dashboard layout).
  2. Automated test and build verification results.
  3. Manual smoke test checklist (verifying login redirect, dashboard display with character cards and encyclopedia grid, role gating differences between PLAYER and DUNGEON_MASTER, clicking into Monsters/NPCs placeholders).
  4. Notice indicating that `add-main-dashboard` is ready for archival via `sdd-spec-archive` and unblocks `prompt-02-add-encyclopedia-browse.md`.
