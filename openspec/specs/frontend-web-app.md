---
id: "frontend-web-app"
title: "Dungeoneer Frontend Web App — Living Specification"
version: "1.2.0"
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
| 3 | **Classe** | Class picker, initial level (1–20), HP calculation method (`MAX`, `AVERAGE`, `ROLLED`), class skills selection, and multiclass level distribution |
| 4 | **Antecedente** | Background picker (filtered by ruleset), ASI (+2/+1 or +1/+1/+1 in 2024), origin feat, and skill collision warning |
| 5 | **Atributos** | Method selector (`Standard Array` / `Point Buy` / `Roll`). Single-chance server roll lock, interactive slot assignment |
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
| **Inventory** | Item list, weight tracker, attunement slots (3 max) | Equip/unequip, add/remove items |
| **Features & Feats** | Active feats grouped by source (class, lineage, background, item) | Collapsible sections |
| **Skills** | All 18 skills with PT-BR names, proficiency/expertise dots, total modifier | Click to roll skill check |
| **Notes** | Persistent character notes list | Create new note, inline editing mode (pencil icon -> editable title & body -> Save/Cancel) and delete modal |

### [RF-04] Server State, Data Fetching & Session Integrity

- **TanStack Query** manages all server state (`characters`, `spells`, `reference` data).
- **Query Keys**: Namespaced as `['characters']`, `['characters', id]`, `['spells', filters]`.
- **Optimistic Updates**: Enabled for HP adjustments and spell slot usage.
- **Cache Stale Time**: `5 minutes` for reference data, `30 seconds` for character data.
- **Error Notifications**: Toast notifications via shadcn/ui `Sonner` consuming RFC 7807 `ProblemDetail` messages.
- **Session Integrity Guard**: `useAuth` executes an asynchronous check against `GET /api/auth/me` on mount. If the backend returns `401 Unauthorized` or `404 Not Found` (e.g. database wipe/docker reset), `clearAuth()` is triggered immediately, purging cookies and `localStorage` to eliminate ghost sessions.
- **PT-BR SRD Translation Layer**: `lib/srd-translations.ts` centralizes translations for abilities, skills, racial traits, equipment proficiencies, and background descriptions applied across all reference hooks.
- **Client-Side Role Gating (`useAuth`)**: The `useAuth` hook exposes reactive `isDM` and `isPlayer` booleans computed from `user.roles` (`DUNGEON_MASTER` / `PLAYER`), with fallback to `false` for unauthenticated sessions.
- **Unified Post-Login Dashboard & Encyclopedia Navigation**: `/characters` serves as the primary hub pós-login containing both the character roster and the `EncyclopediaMenu` component (Arquétipos, Magias, Talentos, Itens, Linhagens, and conditionally Monstros and NPCs for DMs). Monsters and NPCs routes enforce client-side access control with a Dark Fantasy empty-state ("Em breve").
- **Navbar Role Badging**: Displays a gold "Mestre" badge with `Crown` icon next to user identifier when `isDM` is active.

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

---

## Out of Scope

- Native mobile wrappers (iOS/Android).
- Real-time multiplayer canvas/VTT map rendering.

---

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.2.0 | 2026-10-31 | @jaca | Integrated EncyclopediaMenu into post-login dashboard, added client-side role gating (`isDM`/`isPlayer`), Monsters & NPCs DM placeholders, and navbar role badge (`add-main-dashboard`) |
| 1.1.0 | 2026-10-31 | @jaca | Implemented Step 3 multiclass & HP cache, class skills, inline note edit, numeric HP controls, SRD translations, and session integrity guard (`refactor-wizard-hp-srd-and-session`) |
| 1.0.0 | 2026-10-29 | @jaca | Migrated from `docs/spec/res/frontend-spec.md` and `docs/STATE.md` |
