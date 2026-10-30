# Delta Spec: `frontend-web-app`

> **Change ID:** `character-frontend`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

1. **Character Creation Wizard (`[RF-02]`)**:
   - **Step 1 (Name, Ruleset & Initial Level)**: Adds initial level selector (`1–20`) and, when level > 1, HP calculation method selector (`Média`, `Máximo`, `Rolado` with single-chance server roll via `POST /api/characters/hp-rolls`).
   - **Step 2 (Lineage) & Step 4 (Background)**: Displays `SRD_2014` fixed lineage ASI or `SRD_2024` interactive background ASI selector (`+2/+1` or `+1/+1/+1` among 3 allowed abilities) plus informational Origin Feat display.
   - **Step 5 (Ability Scores)**: Supports `Valores Padrão` (`[15, 14, 13, 12, 10, 8]` drag-and-drop), `Compra de Pontos` (27-point budget), and `Rolagem (4d6kh3)` with a single-chance server roll (`POST /api/characters/ability-rolls`) and `@dnd-kit/core` drag-and-drop / swap assignment across the 6 ability slots (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
2. **Character Sheet Layout (`[RF-03]`) — 8-Tab Interface**:
   - Replaces the 8-card grid with an **8-tab navigation layout** at `/characters/[id]`:
     - **Active Tabs with Live Data**: `Resumo` (core stats, HP adjust, AC, initiative, speed, proficiency, saving throws), `Atributos` (6 ability scores & modifiers), `Perícias` (all 18 skills with background proficiencies & calculated modifiers), and `Notas` (editable notes persisted via `PUT /api/characters/{id}`).
     - **Placeholder Tabs ("Em breve")**: `Ações`, `Magias`, `Inventário`, and `Talentos` render themed "Em breve" empty states until their respective backend specs are implemented.
3. **Character Edit (`/characters/[id]/edit`) & Deletion**:
   - Dedicated edit view for `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes` (`PUT /api/characters/{id}`), and physical deletion (`DELETE /api/characters/{id}`) with confirmation modal.
