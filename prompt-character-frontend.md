Focus: Frontend / Next.js 16 / React 19 / TypeScript 5 / TailwindCSS 4 / TanStack Query / @dnd-kit
Task: Execute Full Implementation of Change `character-frontend` in `dungeoneer-frontend`

## Execution Context
- Active Plan: `openspec/changes/character-frontend/tasks.md`
- Linked Proposal: `openspec/changes/character-frontend/proposal.md`
- Delta Spec: `openspec/changes/character-frontend/specs/frontend-web-app-delta.md`
- Target Frontend Directory: `dungeoneer-frontend/`
- Target API Gateway: `process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:9090'`
- Quality & Architecture Contract:
  - Language: User Interface and toast messages strictly in Portuguese (`pt-BR`). Code, variable names, interfaces, comments, and commit messages in English.
  - State Management & API Layer: TanStack Query v5 for server state caching, background invalidations, and optimistic updates; centralized typed API client in `lib/api/`.
  - Dark Fantasy RPG Styling: Adhere to existing Dark Fantasy design system tokens (`bg-slate-950`, `border-amber-900/40`, gold accents, crimson highlights, gothic typography).
  - Drag-and-Drop UX: Implement `@dnd-kit/core` with `PointerSensor` and `KeyboardSensor`, plus tap/click fallback to ensure accessible score assignment across the 6 ability slots (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
  - Defensive Security & RFC 7807: Client-side Zod validation schemas for all forms; automatic `Authorization: Bearer <token>` header attachment; parse RFC 7807 `ProblemDetail` errors into `pt-BR` toasts via `sonner`; route guard on `/characters/**` redirecting unauthenticated users to `/login`.
  - Shift-Left Testing: Vitest + Testing Library tests covering happy paths (TP-01..TP-03) and edge cases/failure modes (TE-01..TE-02).
  - Phase Pauses & Commits: Execute one phase at a time, run local quality gates, present the ready-to-copy Conventional Commit command, update the cumulative walkthrough (`openspec/changes/character-frontend/walkthrough.md` in `pt-BR`), and pause for user approval before advancing to the next phase.

---

### Phase 1: Dependencies, Test Harness Setup & API Client Infrastructure

#### Execution Context
- Scope: Execute Phase 1 only. Install required libraries, configure the Vitest test runner, build the core fetch client with RFC 7807 error parsing, and initialize TanStack Query and Sonner toast providers.

#### Implementation Steps
1. Install dependencies and devDependencies in `dungeoneer-frontend`:
   - Production dependencies: `@tanstack/react-query`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`, `sonner`.
   - Test devDependencies: `vitest`, `@vitejs/plugin-react`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`, `@playwright/test`.
   - Update `package.json` with test scripts: `"test:unit": "vitest run"`, `"test:unit:watch": "vitest"`.
2. Configure Vitest test harness:
   - Create `vitest.config.ts` configured with `@vitejs/plugin-react`, `jsdom` environment, and path alias `@/*` matching `tsconfig.json`.
   - Create `vitest.setup.ts` importing `@testing-library/jest-dom`.
3. Implement API Client in `lib/api/client.ts`:
   - Configurable base URL: `process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:9090'`.
   - Fetch wrapper automatically attaching `Authorization: Bearer <token>` from local storage if present.
   - Parse RFC 7807 `ProblemDetail` responses (`status`, `title`, `detail`, `errors`), converting error payloads into user-friendly `pt-BR` messages.
   - Redirect to `/login` when receiving `401 Unauthorized` responses on authenticated endpoints.
4. Set up Providers:
   - Create `components/providers.tsx` (or update `app/layout.tsx`) wrapping the application with `QueryClientProvider` and `Toaster` (`sonner` with dark fantasy RPG styling).

#### Quality Gates & Verification
- Compile frontend and run Vitest test runner:
  ```bash
  npm --prefix dungeoneer-frontend run build
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/package.json dungeoneer-frontend/package-lock.json dungeoneer-frontend/vitest.config.ts dungeoneer-frontend/vitest.setup.ts dungeoneer-frontend/lib/api/client.ts dungeoneer-frontend/app/layout.tsx dungeoneer-frontend/components/providers.tsx
  git commit -m "build(frontend): add TanStack Query, dnd-kit, sonner, vitest test harness, and core API client"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Create `openspec/changes/character-frontend/walkthrough.md` in Portuguese (`pt-BR`) recording:
  1. Dependencies installed and test harness configuration.
  2. Verification commands and build check results.
  3. Pause and present the commit command to the user before advancing to Phase 2.

---

### Phase 2: Auth State, Gateway Integration & Protected Route Guard

#### Execution Context
- Scope: Execute Phase 2 only. Wire login and registration forms to the backend through Gateway, manage JWT tokens, update Navbar with user state, and establish route protection for `/characters/**`.

#### Implementation Steps
1. Implement Auth Service & Hooks in `lib/api/auth.ts`:
   - Methods: `login(credentials)`, `register(payload)`, `logout()`, `getToken()`, `getUser()`.
   - Implement `useAuth` hook and/or `AuthProvider` to expose current user profile, authentication status, and actions.
   - Handle JWT persistence in `localStorage` securely.
2. Wire Auth UI Components:
   - Update `components/login-form.tsx` to call `POST /api/auth/login`. On success, store token, display success toast, and navigate to `/characters`. On failure, show RFC 7807 error toast.
   - Update `components/register-form.tsx` to call `POST /api/auth/register`. On success, store token and redirect to `/characters`.
3. Update `components/navbar.tsx`:
   - When authenticated: show "Meus Personagens" link, logged-in user nickname/email, and a "Sair" (Logout) button that clears tokens and redirects to `/login`.
   - When unauthenticated: show "Entrar" and "Cadastrar" links.
4. Protect Character Routes:
   - Implement route guard for `/characters/**` (via Next.js Middleware or client-side layout guard in `app/characters/layout.tsx`) that redirects unauthenticated users to `/login`.
   - Preserve public accessibility for `/`, `/login`, `/register`, and `/prototype/character-creation`.

#### Quality Gates & Verification
- Verify linting and build:
  ```bash
  npm --prefix dungeoneer-frontend run lint
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/lib/api/auth.ts dungeoneer-frontend/components/login-form.tsx dungeoneer-frontend/components/register-form.tsx dungeoneer-frontend/components/navbar.tsx dungeoneer-frontend/app/characters/layout.tsx dungeoneer-frontend/middleware.ts
  git commit -m "feat(frontend): integrate auth endpoints, state management, navbar session, and route protection"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. Auth integration and session handling changes.
  2. Route protection verification results.
  3. Pause and present the commit command to the user before advancing to Phase 3.

---

### Phase 3: Character API Hooks & Character List Page (`/characters`)

#### Execution Context
- Scope: Execute Phase 3 only. Implement TanStack Query hooks for characters and reference data, and build the Character List page with responsive Dark Fantasy cards and deletion modal.

#### Implementation Steps
1. Implement Character Query Hooks in `lib/api/characters.ts`:
   - `useCharacters()`: fetches `GET /api/characters` (list of character summaries ordered by `updated_at DESC`).
   - `useCharacter(id)`: fetches `GET /api/characters/{id}` (full character sheet).
   - `useCreateCharacter()`: mutation calling `POST /api/characters`.
   - `useUpdateCharacter()`: mutation calling `PUT /api/characters/{id}`.
   - `useDeleteCharacter()`: mutation calling `DELETE /api/characters/{id}` with query cache invalidation.
   - `useRollAbilityScores()`: mutation calling `POST /api/characters/ability-rolls`.
   - `useRollHitPoints()`: mutation calling `POST /api/characters/hp-rolls`.
   - Reference data hooks: `useReferenceClasses(ruleset)`, `useReferenceLineages(ruleset)`, `useReferenceBackgrounds(ruleset)`.
2. Build Character List Page in `app/characters/page.tsx`:
   - Card grid displaying characters with Dark Fantasy styling tokens: character name, ruleset badge (`5e 2014` / `5e 2024`), level, class, lineage, and current/max HP bar.
   - Quick action buttons on each card:
     - "Abrir Ficha" linking to `/characters/[id]`.
     - "Editar" linking to `/characters/[id]/edit`.
     - "Excluir" opening a confirmation dialog modal.
   - Delete Confirmation Modal: asks the user to confirm deletion; calls `useDeleteCharacter`, displays toast notification, and updates the list.
   - Empty State: themed Dark Fantasy illustration/icon with explanatory message and prominent "Criar Personagem" CTA linking to `/characters/new`.
   - Skeleton loading state and error handling with retry button.

#### Quality Gates & Verification
- Verify linting and build:
  ```bash
  npm --prefix dungeoneer-frontend run lint
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/lib/api/characters.ts dungeoneer-frontend/app/characters/page.tsx dungeoneer-frontend/components/
  git commit -m "feat(frontend): implement character API hooks, character list grid, and deletion confirmation modal"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. Character query hooks and Character List components created.
  2. Build and lint check results.
  3. Pause and present the commit command to the user before advancing to Phase 4.

---

### Phase 4: 6-Step Character Creation Wizard (`/characters/new`)

#### Execution Context
- Scope: Execute Phase 4 only. Elevate the prototype into a production 6-step creation wizard with live reference data, level/HP calculation options, ASI allocation, and `@dnd-kit/core` drag-and-drop attribute assignment.

#### Implementation Steps
1. Migrate and restructure wizard in `app/characters/new/page.tsx`:
   - Reuse visual tokens and Stepper from `app/prototype/character-creation/page.tsx`.
   - Form state managed with React Hook Form + Zod validation.
2. Step 1 — Nome & Regras:
   - Character name (`1–100` characters).
   - Ruleset selector (`SRD_2014` vs `SRD_2024`).
   - Initial Level selector (`1–20`).
   - When level > 1: HP calculation method selector (`MAX`, `AVERAGE`, `ROLLED` with single-chance server roll button calling `useRollHitPoints`).
3. Step 2 — Linhagem:
   - Query `useReferenceLineages(ruleset)`.
   - Display lineage traits, speed, size, and (for `SRD_2014`) fixed ASI bonuses.
4. Step 3 — Classe:
   - Query `useReferenceClasses(ruleset)`.
   - Display hit dice, primary ability, saving throw proficiencies, and spellcaster badge.
5. Step 4 — Antecedente:
   - Query `useReferenceBackgrounds(ruleset)`.
   - Display skill proficiencies.
   - For `SRD_2024`: display Origin Feat informational card and interactive ASI bonus allocator (`+2/+1` or `+1/+1/+1` constrained to the 3 allowed abilities of the selected background).
6. Step 5 — Atributos (3 Methods):
   - `Rolagem (4d6kh3)`:
     - Single-chance "Rolar Atributos" button calling `useRollAbilityScores`.
     - Once rolled, disable the button permanently for this session (persisting token state).
     - Display 6 rolled chips. Drag-and-drop (`@dnd-kit/core`) or tap-to-swap onto the 6 ability slots (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
     - Display base score + ASI bonus = final score and modifier.
   - `Valores Padrão`:
     - Drag-and-drop / tap-to-swap of `[15, 14, 13, 12, 10, 8]` across the 6 ability slots.
   - `Compra de Pontos`:
     - Increment/decrement controls (`8..15`) with real-time 27-point budget counter.
   - Validation Gate: Next button blocked until all 6 ability slots are assigned.
7. Step 6 — Revisão & Creation:
   - Consolidated summary of all choices, derived attributes (HP, AC, Initiative, Speed, Proficiency Bonus, Saving Throws).
   - "Criar Personagem" button calling `useCreateCharacter`.
   - On `201 Created`: redirect to `/characters/[id]` and show success toast.

#### Quality Gates & Verification
- Verify linting and build:
  ```bash
  npm --prefix dungeoneer-frontend run lint
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/characters/new/page.tsx dungeoneer-frontend/components/
  git commit -m "feat(frontend): implement production 6-step character creation wizard with 4d6kh3 drag-and-drop"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. Wizard components, steps, ruleset ASI logic, and dnd-kit integration.
  2. Build and validation status.
  3. Pause and present the commit command to the user before advancing to Phase 5.

---

### Phase 5: 8-Tab Character Sheet View (`/characters/[id]`)

#### Execution Context
- Scope: Execute Phase 5 only. Implement the 8-tab Character Sheet view with live interactive data in `Resumo`, `Atributos`, `Perícias`, and `Notas`, and themed "Em breve" empty states for the remaining tabs.

#### Implementation Steps
1. Build Character Sheet Page in `app/characters/[id]/page.tsx`:
   - Header Section: Character name, level, class, lineage, background, ruleset badge (`5e 2014` / `5e 2024`), `hasPendingLevelChoices` badge (if character was created at level > 1), "Editar" button (`/characters/[id]/edit`), and "Excluir" button with confirmation modal.
2. Implement 8-Tab Navigation:
   - Tab 1 — `Resumo`:
     - Hit Points display (current, max, temp) with quick +/- adjustment buttons calling `useUpdateCharacter`.
     - Combat stats: Armor Class (AC), Initiative modifier, Speed, Proficiency Bonus, Hit Dice.
     - Saving Throws list with proficiency indicators and calculated modifiers.
   - Tab 2 — `Atributos`:
     - 6 ability cards (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`) showing final score, modifier, and saving throw status.
   - Tab 3 — `Perícias`:
     - Complete list of all 18 D&D skills in `pt-BR` (Acrobacia, Adestramento, Arcanismo, etc.).
     - Governing ability score badge, proficiency indicator dot (derived from background/class), and total calculated skill modifier.
   - Tab 4 — `Notas`:
     - Full-featured notes textarea for character backstory, session notes, and personality traits.
     - Save button and/or debounced auto-save triggering `useUpdateCharacter`.
   - Tab 5 — `Ações`: Themed Dark Fantasy "Em breve" empty state.
   - Tab 6 — `Magias`: Themed Dark Fantasy "Em breve" empty state.
   - Tab 7 — `Inventário`: Themed Dark Fantasy "Em breve" empty state.
   - Tab 8 — `Talentos`: Themed Dark Fantasy "Em breve" empty state (with informational card for 2024 Origin Feat if present).

#### Quality Gates & Verification
- Verify linting and build:
  ```bash
  npm --prefix dungeoneer-frontend run lint
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/characters/[id]/page.tsx dungeoneer-frontend/components/
  git commit -m "feat(frontend): implement 8-tab character sheet view with live stats, skills, hp adjust, and notes"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. 8-tab sheet architecture, live tabs, and placeholder tabs.
  2. Build and UI verification results.
  3. Pause and present the commit command to the user before advancing to Phase 6.

---

### Phase 6: Character Edit Page (`/characters/[id]/edit`)

#### Execution Context
- Scope: Execute Phase 6 only. Implement the dedicated Character Edit form allowing updates to mutable fields, enforcing domain immutability, and providing character deletion.

#### Implementation Steps
1. Build Character Edit Page in `app/characters/[id]/edit/page.tsx`:
   - Form managed with React Hook Form + Zod schema validation.
   - Editable fields:
     - `name`: 1–100 characters, required.
     - `hitPointsCurrent`: integer between 0 and `hitPointsMax`.
     - `hitPointsTemp`: integer >= 0.
     - `notes`: text string.
   - Read-only summary cards: display immutable character properties (ruleset, level, class, lineage, background, base ability scores) with clear indication that they cannot be modified via sheet edit.
2. Actions & Navigation:
   - "Salvar Alterações" button: submits `PUT /api/characters/{id}` via `useUpdateCharacter`, invalidates query cache, displays success toast, and navigates back to `/characters/[id]`.
   - "Cancelar" button: navigates back to `/characters/[id]` without changes.
   - "Excluir Personagem" button: opens confirmation modal calling `useDeleteCharacter` and redirects to `/characters`.

#### Quality Gates & Verification
- Verify linting and build:
  ```bash
  npm --prefix dungeoneer-frontend run lint
  npm --prefix dungeoneer-frontend run build
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/app/characters/[id]/edit/page.tsx
  git commit -m "feat(frontend): implement character edit page with zod validation and delete confirmation"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. Character edit page implementation and validation rules.
  2. Build verification results.
  3. Pause and present the commit command to the user before advancing to Phase 7.

---

### Phase 7: Shift-Left Automated Test Suite (Vitest & Testing Library)

#### Execution Context
- Scope: Execute Phase 7 only. Implement unit and component tests verifying happy path scenarios (TP-01..TP-03) and edge cases/failure modes (TE-01..TE-02).

#### Implementation Steps
1. Implement Happy Path Tests (`dungeoneer-frontend/__tests__/`):
   - TP-01 (`character-wizard.test.tsx`):
     - Renders 6 steps sequentially.
     - Handles Point Buy 27-point calculations.
     - Handles Standard Array assignment.
     - Handles single-chance `4d6kh3` roll and drag-and-drop / swap assignment.
     - Submits `POST /api/characters` with expected payload.
   - TP-02 (`character-sheet.test.tsx`):
     - Renders all 8 tabs.
     - Verifies tab switching and live data in `Resumo` (HP, AC, Initiative), `Atributos` (6 modifiers), `Perícias` (18 skills with proficiencies), and `Notas` (editing and saving).
     - Verifies "Em breve" placeholder states in `Ações`, `Magias`, `Inventário`, and `Talentos`.
   - TP-03 (`character-list-and-edit.test.tsx`):
     - Renders character card grid from API.
     - Verifies delete confirmation modal and `DELETE /api/characters/{id}` invocation.
     - Verifies edit form loads character data and submits `PUT /api/characters/{id}`.
2. Implement Edge Cases & Failure Mode Tests:
   - TE-01 (`wizard-ability-roll-guard.test.tsx`):
     - Verifies "Rolar Atributos" button cannot be clicked a second time once rolled.
     - Verifies wizard blocks advancing to Step 6 if any of the 6 ability slots remains unassigned.
   - TE-02 (`problem-detail-toast.test.tsx`):
     - Verifies RFC 7807 `ProblemDetail` errors (e.g., 409 Conflict, 422 Unprocessable Entity) trigger a user-facing toast in `pt-BR`.

#### Quality Gates & Verification
- Run Vitest unit and component test suite:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add dungeoneer-frontend/__tests__/
  git commit -m "test(frontend): implement comprehensive Vitest component and unit test suite"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
  1. Test suite coverage (TP-01..TP-03 and TE-01..TE-02).
  2. Vitest test execution metrics and results.
  3. Pause and present the commit command to the user before advancing to Phase 8.

---

### Phase 8: Full Verification Gate, Spec Deltas & Final Walkthrough Consolidation

#### Execution Context
- Scope: Execute Phase 8. Run the complete build and test suite, update delta specifications, complete the task checklist, and produce the final consolidated walkthrough.

#### Implementation Steps
1. Run Full Verification Gate:
   ```bash
   npm --prefix dungeoneer-frontend run lint
   npm --prefix dungeoneer-frontend run build
   npm --prefix dungeoneer-frontend run test:unit
   ```
2. Update Specification Delta:
   - Review and finalize `openspec/changes/character-frontend/specs/frontend-web-app-delta.md` ensuring full documentation of:
     - 8-tab character sheet structure.
     - Single-chance `4d6kh3` drag-and-drop assignment in Step 5.
     - Level `1–20` and HP calculation method selection in Step 1.
3. Update Tasks Checklist:
   - Mark all tasks in `openspec/changes/character-frontend/tasks.md` as completed (`[x]`).
4. Finalize Cumulative Walkthrough in `openspec/changes/character-frontend/walkthrough.md` (`pt-BR`):
   - Consolidate deliverables across all 8 phases.
   - Include test results and execution summary.
   - Provide manual testing verification steps (e.g., starting backend, logging in, creating character, viewing 8-tab sheet, editing, and deleting).

#### Quality Gates & Verification
- Verify clean status and passing tests:
  ```bash
  npm --prefix dungeoneer-frontend run test:unit
  ```
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/character-frontend/ prompt-character-frontend.md
  git commit -m "chore(frontend): complete verification, delta spec documentation, and final walkthrough"
  ```

#### Final Phase Handoff
- Present the consolidated walkthrough to the user and notify readiness for `sdd-spec-validate` or `sdd-spec-archive`.
