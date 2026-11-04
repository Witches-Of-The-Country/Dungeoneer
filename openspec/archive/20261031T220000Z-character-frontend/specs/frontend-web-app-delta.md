# Delta Spec: `frontend-web-app`

> **Change ID:** `character-frontend`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

1. **Application Routes & Route Protection (`[RF-01]`)**:
   - Marks `/characters`, `/characters/new`, `/characters/[id]`, and `/characters/[id]/edit` as **Implemented**.
   - Enforces route protection on `/characters/**` via Next.js Middleware (`middleware.ts`) and client-side layout guard (`app/characters/layout.tsx`), redirecting unauthenticated users to `/login`.
   - Connects `/login`, `/register`, and `Navbar` to `POST /api/auth/login` and `POST /api/auth/register` through `api-gateway` (`:9090`), persisting JWT tokens and attaching `Authorization: Bearer <token>` on authenticated requests.

2. **Character Creation Wizard (`[RF-02]`) — `/characters/new`**:
   - **Step 1 (Nome & Regras)**: Character name (`1–100` characters, Zod-validated), ruleset selector (`SRD_2014` vs `SRD_2024`), initial level selector (`1–20`), and, when level > 1, HP calculation method selector (`MAX`, `AVERAGE`, `ROLLED` with single-chance server roll via `POST /api/characters/hp-rolls`).
   - **Step 2 (Linhagem)**: Queries `GET /api/reference/lineages?ruleset=...`, displaying racial traits, speed, size, and (for `SRD_2014`) fixed lineage ASI bonuses.
   - **Step 3 (Classe)**: Queries `GET /api/reference/classes?ruleset=...`, displaying hit dice, primary ability, saving throw proficiencies, and spellcaster indicator.
   - **Step 4 (Antecedente)**: Queries `GET /api/reference/backgrounds?ruleset=...`, displaying skill proficiencies, and (for `SRD_2024`) informational Origin Feat card plus interactive ASI allocator (`+2/+1` or `+1/+1/+1` constrained to the 3 allowed abilities of the selected background).
   - **Step 5 (Atributos)**: Supports 3 generation methods:
     - `Rolagem (4d6kh3)`: Single-chance server roll (`POST /api/characters/ability-rolls`) that locks the roll button after one use (persisted in `sessionStorage`) and allows assigning the 6 rolled values via `@dnd-kit/core` drag-and-drop (`PointerSensor` + `KeyboardSensor`) or tap/click onto the 6 ability slots (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
     - `Valores Padrão`: Drag-and-drop / tap assignment of `[15, 14, 13, 12, 10, 8]` across the 6 ability slots.
     - `Compra de Pontos`: Increment/decrement controls (`8..15`) with live `27`-point budget counter.
     - Progression to Step 6 is blocked until all 6 ability slots are assigned.
   - **Step 6 (Revisão & Criação)**: Summarizes identity, derived stats (HP, AC, Initiative, Proficiency Bonus), and final ability scores, submitting `POST /api/characters` and redirecting to `/characters/[id]`.

3. **Character Sheet Layout (`[RF-03]`) — 8-Tab Interface at `/characters/[id]`**:
   - Replaces the 8-card grid with an **8-tab navigation layout**:
     - **Active Tabs with Live Data**:
       - `Resumo`: Core stats, current/max/temp HP with quick `+/-` adjustment (`PUT /api/characters/{id}`), AC, Initiative, Speed, Proficiency Bonus, Hit Dice, and Saving Throws with proficiency indicators.
       - `Atributos`: 6 ability cards (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`) with final score, calculated modifier, and saving throw proficiency indicator.
       - `Perícias`: All 18 D&D skills in `pt-BR`, showing governing ability badge, proficiency dot (derived from background), and calculated total modifier.
       - `Notas`: Editable textarea for character notes/backstory persisted via `PUT /api/characters/{id}`.
     - **Placeholder Tabs ("Em breve")**:
       - `Ações`, `Magias`, `Inventário`, and `Talentos` render themed Dark Fantasy "Em breve" empty states (`Talentos` also displays the 2024 Origin Feat card when applicable).

4. **Character List (`/characters`), Edit (`/characters/[id]/edit`) & Deletion**:
   - `/characters`: Responsive card grid ordered by `updated_at DESC` with ruleset badge, level/class/lineage, HP bar, empty state CTA, and quick actions (`Abrir Ficha`, `Editar`, `Excluir`).
   - `/characters/[id]/edit`: React Hook Form + Zod form for updating mutable fields (`name`, `hitPointsCurrent`, `hitPointsTemp`, `notes`) via `PUT /api/characters/{id}` alongside read-only summary cards of immutable creation properties.
   - Physical deletion (`DELETE /api/characters/{id}`) guarded by a confirmation modal (`DeleteCharacterModal`).
