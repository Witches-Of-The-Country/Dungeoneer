---
id: "frontend-web-app"
title: "Dungeoneer Frontend Web App — Living Specification"
version: "1.11.0"
status: "active"
owners:
  - "@jaca"
scope:
  paths:
    - "dungeoneer-frontend/**"
resources:
  - "openspec/specs/system-architecture.md"
  - "openspec/specs/player-character-domain.md"
---

# Dungeoneer Frontend Web App — Living Specification

> This is the authoritative source of truth for the `dungeoneer-frontend` Next.js 16 application, Dark Fantasy RPG design system, routes, Character Creation Wizard, Character Sheet Dashboard, and state management conventions.
> Update this file only after merging a completed change from `openspec/changes/`.

---

## Context & Objectives

`dungeoneer-frontend` delivers a responsive, Dark Fantasy RPG user interface built with Next.js 16 (App Router), React 19, TypeScript 5, TailwindCSS 4, shadcn/ui + Radix UI primitives, TanStack Query, React Hook Form + Zod, Lucide React icons, and `next-themes`. All user-facing text and labels are localized in **Portuguese (`pt-BR`)**.

---

## Functional Requirements

### [RF-01] Application Routes (Phase 1)

| Route | Description | Status |
|-------|-------------|--------|
| `/` | Landing page with sticky global Navbar | Implemented |
| `/login` | User authentication (Dark Fantasy UI, Zod + React Hook Form) | Implemented |
| `/register` | User registration (Zod + React Hook Form validation) | Implemented |
| `/characters` | Main unified dashboard (Hero cards list + Encyclopedia & Reference menu) | Implemented |
| `/characters/new` | Character creation wizard (6-step flow with multiclass & translations) | Implemented |
| `/characters/[id]` | Character sheet dashboard (8-card layout, inline note edit, numeric HP adjustments) | Implemented |
| `/characters/[id]/edit` | Edit character details | Implemented |
| `/characters/[id]/level-up` | Level-up flow | Planned |
| `/encyclopedia/monsters` | Bestiary & Monsters placeholder (DM-only empty state "Em breve" + route guard) | Implemented |
| `/encyclopedia/npcs` | NPC Gallery placeholder (DM-only empty state "Em breve" + route guard) | Implemented |
| `/spells` | Spell search & filter page | Planned |

### [RF-02] Character Creation Wizard (6-Step Flow)

| Step | Title | Content |
|------|-------|---------|
| 1 | **Nome & Regras** | Character name input, ruleset selector (`SRD_2014` vs `SRD_2024`) |
| 2 | **Linhagem** | Lineage picker (filtered by ruleset), preview granted traits with PT-BR translations |
| 3 | **Classe** | Class picker, initial level (1–20) with decoupled numeric input allowing intermediate string editing and onBlur clamping to `[1, 20]`, HP calculation method (`MAX`, `AVERAGE`, `ROLLED`), class skills selection, and multiclass level distribution |
| 4 | **Antecedente** | Background picker (filtered by ruleset), ASI (+2/+1 or +1/+1/+1 in 2024), origin feat (strictly filtered to `featType === 'ORIGIN'`), and skill collision warning |
| 5 | **Atributos** | Method selector (`Standard Array` / `Point Buy` / `Roll`). Single-chance server roll lock, interactive slot assignment, and creation re-roll support via `forceNew=true` (invalidating previous unused rolls) |
| 6 | **Revisão & Criação** | Full character summary with live derived stats (HP, AC, Initiative, Speed, PB), confirm and submit (`POST /api/characters`) |

**Step 3 Mechanics & Multiclass HP Rules:**
- **Initial Level & Multiclass Distribution:** Primary class has base level 1. If total level > 1, secondary classes can be added with independent level allocations. Validation enforces that the sum of class levels exactly matches the chosen total character level.
- **HP Rolling Strategy:**
  - *Mono-class:* Level 1 receives max hit die; levels 2–20 roll $(level - 1)$ dice.
  - *Multiclass:* Primary class receives max hit die at level 1 and rolls $(primaryLevel - 1)$ dice if $primaryLevel > 1$. Secondary class at level $Y$ rolls $Y$ dice (no max hit die).
- **Per-Class HP Cache:** Rolled HP tokens and dice results are cached by `classId` in component state (`hpRollsByClass`) ensuring selections are preserved during class toggles.
- **Class Skill Selection:** Player selects allowed skills up to the class limit. Step 4 detects collisions with background skills, offering immediate replacement.

### [RF-03] Character Sheet Dashboard (Card-Based Layout)

Responsive grid (3 columns on desktop, 2 on tablet, 1 on mobile) with collapsible/expandable cards (persisting state in `localStorage`):

| Card | Content | Interactive Behavior |
|------|---------|----------------------|
| **Core Stats / PV** | Name, level, classes, lineage, HP bar, AC, initiative, proficiency bonus, speed | Micro-adjustments (`-1 PV`, `+1 PV`) and numeric input controls (Damage `-` and Healing `+`) with auto-resetting inputs |
| **Ability Scores** | 6 stats in 2×3 grid with PT-BR labels (`Força`, `Destreza`, etc.). Score + modifier + saving throw dot | Click to roll ability check |
| **Actions & Attacks** | Weapon attacks and cantrips. To-hit bonus, damage formula | Click to roll attack/damage |
| **Spellcasting** | Spell slot tracker (pip-style), spell list by level, prepared/known filter | Click slots to use/recover, click spell to roll |
| **Inventory** | Item list, weight tracker, attunement slots (3 max), Item Catalog & Store modal with canonical category tabs (WEAPON, ARMOR, POTION, WONDROUS, RING, SCROLL, WAND, ROD, STAFF, EQUIPMENT) and copper coin purchasing logic | Equip/unequip, add/remove items, browse store |
| **Features & Feats** | Active feats grouped by source (class, lineage, background, item) | Collapsible sections |
| **Skills** | All 18 skills with PT-BR names, proficiency/expertise dots, total modifier | Click to roll skill check |
| **Notes** | Persistent character notes list | Create new note, inline editing mode (pencil icon -> editable title & body -> Save/Cancel) and delete modal |

### [RF-04] Server State, Data Fetching & Session Integrity

- **TanStack Query** manages all server state (`characters`, `spells`, `reference` data).
- **Query Keys**: Namespaced as `['characters']`, `['characters', id]`, `['spells', filters]`.
- **Optimistic Updates**: Enabled for HP adjustments and spell slot usage.
- **Cache Stale Time**: `5 minutes` for reference data, `30 seconds` for character data.
- **Error Notifications**: Toast notifications via shadcn/ui `Sonner` consuming RFC 7807 `ProblemDetail` messages.
- **HttpOnly Session & Token Management**: `dungeoneer_auth_token` is synchronized in cookies and `localStorage` upon authentication so Next.js Edge Middleware (`middleware.ts`) can validate protected routes server-side (`/characters`, `/profile`). All `apiFetch` and auth calls use `credentials: 'include'`, with `getAccessToken()` serving as the single token provider for `Authorization: Bearer`.
- **Session Integrity & Rehydration**: On mount/refresh, `useAuth` restores the session from in-memory token, cookies, or `localStorage`, verifying user profile via `GET /api/auth/me`. If the backend returns `401 Unauthorized` or `404 Not Found`, `clearAuth()` purges tokens and cookies to prevent ghost sessions.
- **Loop-Free Redirect Guard**: `middleware.ts` clears invalid or expired `dungeoneer_auth_token` cookies when redirecting to `/login?redirect=`, and `LoginForm` clears stale sessions once on initial mount without prematurely logging out newly authenticated submissions. Safe redirect targets enforce relative paths (`/` and `!//`). `RegisterForm` awaits a confirmed session before navigating to `/characters`.
- **PT-BR SRD Translation Layer**: `lib/srd-translations.ts` centralizes translations for abilities, skills, racial traits, equipment proficiencies, and background descriptions applied across all reference hooks.
- **Client-Side Role Gating (`useAuth`)**: The `useAuth` hook exposes reactive `isDM` and `isPlayer` booleans computed from `user.roles` (`DUNGEON_MASTER` / `PLAYER`), with fallback to `false` for unauthenticated sessions.
- **Unified Post-Login Dashboard & Encyclopedia Navigation**: `/characters` serves as the primary hub pós-login containing both the character roster and the `EncyclopediaMenu` component (Arquétipos, Magias, Talentos, Itens, Linhagens, and conditionally Monstros and NPCs for DMs). Monsters and NPCs routes enforce client-side access control with a Dark Fantasy empty-state ("Em breve").
- **Navbar Navigation & Role Badging**: Dynamically points the brand logo link to `/characters` (Dashboard) when authenticated and `/` when unauthenticated. Displays a gold "Mestre" badge with `Crown` icon next to the user identifier when `isDM` is active.

---

## Non-Functional Requirements

### [RNF-01] Dark Fantasy RPG Design System

| Token | Value | Usage |
|-------|-------|-------|
| Background (base) | `#0f0f1a` range | Page backgrounds |
| Background (card) | Parchment-toned with subtle texture | Card surfaces |
| Accent (primary) | Warm gold/amber `hsl(40, 90%, 55%)` range | Buttons, links, highlights |
| Accent (danger) | Crimson | HP loss, delete actions |
| Accent (success) | Emerald | HP gain, level-up confirmations |

### [RNF-02] Typography

| Role | Font | Weight |
|------|------|--------|
| Headings (`h1`–`h3`) | `Cinzel` | `700` (bold) |
| Body text | `Inter` | `400` (regular), `500` (medium) |
| Monospace / stats | `JetBrains Mono` | `400` |

### [RNF-03] Micro-Animations

- **Dice roll**: 3D tumbling animation on click (client-side CSS + JS).
- **Card hover**: Subtle lift + border glow.
- **HP bar**: Smooth transition on value change.
- **Level-up**: Celebratory particle burst.
- **Page transitions**: Fade-in with slight upward slide.

### [RNF-04] Responsive Breakpoints

| Breakpoint | Layout |
|-----------|--------|
| `≥1280px` (`xl`) | 3-column card grid, full sidebar |
| `≥768px` (`md`) | 2-column card grid, collapsed sidebar |
| `<768px` (`sm`) | Single-column stack, bottom nav |

### [RF-03] Level-Up Wizard Modal (`LevelUpWizardModal`)

- **Class & Subclass (Step 1)**: Allows existing class advancement or unrestricted multiclass leveling with subclass gating.
- **Hit Points (Step 2)**: Locked to character's career HP method (`MAX`, `AVERAGE`, `ROLLED` with server-authority single-chance token).
- **ASI vs Feat (Step 3)**: Unlocked at ASI levels. When "Talento" is chosen:
  - Renders rich feat catalog search and filter chips (`ALL`, `ORIGIN`, `GENERAL`, `HOMEBREW`).
  - **Half-Feat Support**: Automatically detects `mechanicalEffects.abilityBonusChoice`. Renders an interactive sub-panel ("Bônus de Atributo do Talento (+1)") with selectable chips for allowed abilities (`allowedAbilities`), displaying current and projected ability scores (+1, capped at 20).
  - **Gating**: Blocks progression to Step 4 if an ability choice is required by a half-feat but not yet selected.
- **Review & Submit (Step 4)**: Summarizes class level, HP increase, and chosen feat/ability increase (e.g., `Talento: Atleta (+1 em Força)`), dispatching `abilityScoreIncreases: { [chosenStat]: 1 }` alongside `selectedFeatId`.

### [RF-04] Homebrew Feat Charges & Character Sheet Feats Tab

- **Homebrew Charge Builder (`CreateHomebrewDialog`)**:
  - Interactive charge section with toggle "Habilitar Sistema de Cargas e Descanso".
  - Formula selection: `FIXED`, `ABILITY_MODIFIER` (with ability picker), `PROFICIENCY`, `LEVEL`.
  - Independent rest recovery rules:
    - Short rest: `NONE` (default), `ALL`, `FIXED`, `MODIFIER`, `PROFICIENCY`, `LEVEL`.
    - Long rest: `ALL` (default), `NONE`, `FIXED`, `MODIFIER`, `PROFICIENCY`, `LEVEL`.
  - Full edit hydration restoring existing mechanical effects and legacy fallbacks.
- **Feats Tab & Modals (`FeatsTab`, `FeatDetailModal`)**:
  - Live charge counter `{currentCharges} / {maxCharges}` with single-click consumption (`Zap` button).
  - Formula badges (e.g., `Fórmula: Mod. SAB`) and rest recovery tags (e.g., `Descanso Curto: Total`, `Descanso Longo: Total`).
  - Active mechanics breakdown in `FeatDetailModal` reflecting charges and rest recovery.

### [RF-05] Rest System Modal & Sheet Controls (`RestModal`)

- **Sheet Header Controls**:
  - "Descanso Curto" and "Descanso Longo" action buttons in the character sheet header.
  - Multi-class Hit Dice counters displaying `{availableHitDice}/{classLevel} ({hitDie})`.
  - Conditional exhaustion badge rendered when `exhaustion > 0` (e.g., `Exaustão: 2`).
- **Rest Modal (`RestModal`)**:
  - Mode selector toggling between Short Rest and Long Rest.
  - Guard: blocks rest submission if the character is unconscious or dead (`hitPointsCurrent === 0`).
  - **Short Rest Mode**:
    - Stepper inputs per class progression to choose number of Hit Dice to spend (`0` to `availableHitDice`).
    - Visual feedback showing the character's CON modifier added per die rolled.
  - **Long Rest Mode**:
    - Explanatory confirmation screen detailing recovery of all HP, half total Hit Dice, spell slots, long-rest feat charges, and reduction of 1 exhaustion level.
  - Integrated with `useTakeRest` mutation (`POST /api/characters/{id}/rest`), automatically invalidating the character sheet cache on success and displaying RFC 7807 toast alerts on error.

### [RF-06] Charged Actions & Dice Roller Deduction (`ActionsTab`, `DiceRollerModal`)

- **Charged Action Button Visibility**: In `ActionsTab`, any feat action configured with `maxCharges > 0` displays the "Gastar" action button, even if `isRollable` is true.
- **Roll Consumption Integration**: When rolling an action associated with a charged feat (`maxCharges > 0` and `currentCharges > 0`), `DiceRollerModal` automatically executes `useFeatMutation` to spend 1 charge on roll trigger, keeping the character's active charge counters synchronized with dice rolls.

### [RF-07] Equipment Slots Grid, Temp HP Absorption & Custom Actions CRUD

- **Equipment Slots Overview (`InventoryTab`)**:
  - Displays the 12 canonical equipment slots (`Peito`, `Capacete`, `Luvas`, `Botas`, `Calças`, `Capa`, `Colar`, `Acessório / Cinto`, `Mão Direita`, `Mão Esquerda`, `Anel 1`, `Anel 2`) in a responsive grid.
  - Slots display the equipped item name, item rarity badge, slot name, and a direct "Tirar" action button. Empty slots display a dashed placeholder.
  - Equipping an item automatically assigns it to its default or compatible slot, resolving slot occupancy on the backend.
- **Combat Sheet Temporary HP Absorption (`app/characters/[id]/page.tsx`)**:
  - Incoming damage applied via `handleApplyDamage` or quick-decrement `-1 PV` absorbs from `hitPointsTemp` first before spilling over to `hitPointsCurrent`.
  - Damage adjustment buttons remain interactable whenever `hitPointsCurrent > 0` OR `hitPointsTemp > 0`.
- **Custom Actions Management (`ActionsTab`)**:
  - Custom combat actions (`category === 'CUSTOM'`) provide full CRUD controls: creation modal, edit modal with live field pre-population, and deletion confirmation modal.

---

## Out of Scope

- Native mobile wrappers (iOS/Android).
- Real-time multiplayer canvas/VTT map rendering.

---

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.11.0 | 2026-11-05 | @jaca | Documented wizard origin feat filtering, level input decoupling onBlur clamping, creation ability reroll, and login redirect loop fix (`004-fix-character-creation-and-reroll-rules`) |
| 1.10.0 | 2026-11-05 | @jaca | Documented 12th TRINKET slot (Acessório / Cinto) in InventoryTab equipment grid and test-ids (`007-add-trinket-equipment-slot`) |
| 1.9.0 | 2026-11-05 | @jaca | Documented 11-slot equipment grid in InventoryTab, Temp HP damage absorption order in Character Sheet combat controls, and Custom Actions Edit/Delete modals in ActionsTab (`003-fix-equipment-slots-and-combat-sheet`) |
| 1.8.0 | 2026-11-05 | @jaca | Documented charged actions button display and automatic charge deduction upon dice roll in ActionsTab and DiceRollerModal (`002-fix-legacy-items-and-feats-mechanics`) |
| 1.7.0 | 2026-11-05 | @jaca | Consolidated HttpOnly session model, credentials inclusion, `/api/auth/me` rehydration, loop-free redirect guard, and dynamic Navbar brand link (`001-fix-auth-post-register-loop`) |
| 1.6.0 | 2026-11-03 | @jaca | Added RestModal, header rest buttons, multi-class Hit Dice counters, and exhaustion badge (`add-long-short-rest`) |
| 1.5.0 | 2026-11-03 | @jaca | Added Homebrew Feat charge builder form, independent rest recovery selectors, and Feats tab charge/recovery indicators (`add-homebrew-feat-charges`) |
| 1.4.0 | 2026-11-03 | @jaca | Added Level-Up Wizard half-feat ability score selection widget, validation gating, and payload dispatch (`level-up-half-feat-ability-choice`) |
| 1.3.0 | 2026-11-02 | @jaca | Aligned encyclopedia and inventory item catalog filters with canonical categories and rarities (`008-srd-item-catalog-seed-and-category-normalization`) |
| 1.2.0 | 2026-10-31 | @jaca | Integrated EncyclopediaMenu into post-login dashboard, added client-side role gating (`isDM`/`isPlayer`), Monsters & NPCs DM placeholders, and navbar role badge (`add-main-dashboard`) |
| 1.1.0 | 2026-10-31 | @jaca | Implemented Step 3 multiclass & HP cache, class skills, inline note edit, numeric HP controls, SRD translations, and session integrity guard (`refactor-wizard-hp-srd-and-session`) |
| 1.0.0 | 2026-10-29 | @jaca | Migrated from `docs/spec/res/frontend-spec.md` and `docs/STATE.md` |
