---
id: "frontend-web-app"
title: "Dungeoneer Frontend Web App — Living Specification"
version: "1.0.0"
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
| `/characters` | Character list (grid/table view) | Planned |
| `/characters/new` | Character creation wizard (6-step flow) | Planned |
| `/characters/[id]` | Character sheet dashboard (8-card layout) | Planned |
| `/characters/[id]/edit` | Edit character details | Planned |
| `/characters/[id]/level-up` | Level-up flow | Planned |
| `/spells` | Spell search & filter page | Planned |

### [RF-02] Character Creation Wizard (6-Step Flow)

| Step | Title | Content |
|------|-------|---------|
| 1 | **Name & Ruleset** | Character name input, ruleset selector (5e 2014 / 2024) |
| 2 | **Lineage** | Lineage/Race picker (filtered by ruleset), preview granted feats |
| 3 | **Class** | Class picker, hit dice preview, spellcasting type indicator |
| 4 | **Background** | Background picker (filtered by ruleset), preview skills/feats |
| 5 | **Ability Scores** | Method selector (`Standard Array` / `Point Buy` / `Roll`). Score assignment UI with modifier preview |
| 6 | **Review & Create** | Full character summary, confirm and submit (`POST /api/characters`) |

**Ability Score Generation UI Modes:**
- **Standard Array**: Drag-and-drop `15, 14, 13, 12, 10, 8` to ability slots.
- **Point Buy**: `+/-` buttons per ability with running point budget counter (`27` max).
- **Rolled**: Server-validated `4d6-drop-lowest`. "Roll" button generates 6 values for user assignment.

### [RF-03] Character Sheet Dashboard (Card-Based Layout)

Responsive grid (3 columns on desktop, 2 on tablet, 1 on mobile) with collapsible/expandable cards (persisting state in `localStorage`):

| Card | Content | Interactive Behavior |
|------|---------|----------------------|
| **Core Stats** | Name, level, class(es), lineage, HP bar (current/max/temp), AC, initiative, proficiency bonus, speed | Click HP to adjust, toggle conditions |
| **Ability Scores** | 6 stats in 2×3 grid. Score + modifier + saving throw proficiency dot | Click to roll ability check |
| **Actions & Attacks** | Weapon attacks and cantrips. To-hit bonus, damage formula | Click to roll attack/damage |
| **Spellcasting** | Spell slot tracker (pip-style), spell list by level, prepared/known filter | Click slots to use/recover, click spell to roll |
| **Inventory** | Item list, weight tracker, attunement slots (3 max) | Equip/unequip, add/remove items |
| **Features & Feats** | Active feats grouped by source (class, lineage, background, item) | Collapsible sections |
| **Skills** | All 18 skills, proficiency/expertise dots, total modifier | Click to roll skill check |
| **Notes** | Free-form text: backstory, personality traits, ideals, bonds, flaws | Editable text areas |

### [RF-04] Server State & Data Fetching

- **TanStack Query** manages all server state (`characters`, `spells`, `reference` data).
- **Query Keys**: Namespaced as `['characters']`, `['characters', id]`, `['spells', filters]`.
- **Optimistic Updates**: Enabled for HP adjustments and spell slot usage.
- **Cache Stale Time**: `5 minutes` for reference data, `30 seconds` for character data.
- **Error Notifications**: Toast notifications via shadcn/ui `Sonner` consuming RFC 7807 `ProblemDetail` messages.

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
| 1.0.0 | 2026-10-29 | @jaca | Migrated from `docs/spec/res/frontend-spec.md` and `docs/STATE.md` |
