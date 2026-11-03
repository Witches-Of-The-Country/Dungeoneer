Focus: Backend / Spring Boot 4 (Java 21) / Flyway Database Migration / PostgreSQL 15
Task: Execute Phase 1: Database Migration V26__seed_complete_srd_general_and_origin_feats.sql

## Execution Context
- Active Plan: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/tasks.md`
- Linked Proposal: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/proposal.md`
- Data Model Delta: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/specs/player-character-data-model-delta.md`
- Scope: Execute Phase 1 only (Tasks 1.1 to 1.6). Inspect, validate, and verify Flyway migration `V26__seed_complete_srd_general_and_origin_feats.sql`, execute backend migration tests, and initialize the cumulative walkthrough artifact. Do not proceed to subsequent phases.

## Implementation Steps
1. **Inspect and Validate Flyway Migration V26:**
   - Inspect `player-character-service/backend/src/main/resources/db/migration/V26__seed_complete_srd_general_and_origin_feats.sql`.
   - Verify deterministic UUID ranges: `30000000-0000-0000-0000-0000000007xx` for SRD 2014 General Feats, `40000000-0000-0000-0000-0000000007xx` for SRD 2024 General Feats, and `40000000-0000-0000-0000-0000000008xx` for SRD 2024 Origin Feats.
   - Confirm missing iconic SRD 2014 General Feats (40+ feats: Alert, Athlete, Actor, War Caster, Crossbow Expert, Great Weapon Master, Shield Master, Polearm Master, Sentinel, Sharpshooter, Lucky, Tough, etc.) are present with localized Portuguese names and rules text.
   - Confirm missing SRD 2024 General Feats (30+ feats: Athlete, Charger, Crossbow Expert, Defensive Duelist, Dual Wielder, Great Weapon Master, Polearm Master, Sentinel, Sharpshooter, Skill Expert, etc.) and Origin Feats (*Crafter, Healer, Lucky, Musician, Tavern Brawler*) are properly seeded.
2. **Verify Normalization & Mechanical Effects Payloads:**
   - Confirm SQL UPDATE statements normalize legacy V14 entries (*Magic Initiate / Iniciado em Magia*, *Savage Attacker / Atacante Selvagem*, *Skilled / Habilidoso*) with `feat_type = 'ORIGIN'`.
   - Ensure structured `mechanical_effects` JSON payloads contain prerequisite rules, ability bonus choices, and `spellChoice` metadata (`cantripsCount`, `spellsCount`, `spellLevel`, `allowedClasses`) for *Magic Initiate* and *Ritual Caster*.
   - Verify idempotency using `ON CONFLICT (ruleset, slug) DO UPDATE`.
3. **Execute Backend Migration & Integration Tests:**
   - Run Maven test suite on `player-character-service` backend using global `mvn` to verify PostgreSQL test containers apply migration `V26` with zero SQL syntax errors or constraint violations.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(feats): seed complete srd general and origin feats via migration V26"
  ```

## Phase Handoff & Cumulative Walkthrough
Create the shared walkthrough artifact (`openspec/changes/009-seed-complete-srd-general-and-origin-feats/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results (Flyway migration output and backend test execution).
3. Handoff context prepared for Phase 2 (Backend Persistence & Domain Support).
Pause and prompt the user to commit progress before proceeding to Phase 2.
