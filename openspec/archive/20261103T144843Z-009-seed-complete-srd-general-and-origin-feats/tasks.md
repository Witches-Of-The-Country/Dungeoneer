# Tasks: 009-seed-complete-srd-general-and-origin-feats

> **Change ID:** `009-seed-complete-srd-general-and-origin-feats`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Phase 1: Database Migration `V26__seed_complete_srd_general_and_origin_feats.sql`
- [x] 1.1 Draft migration script `V26__seed_complete_srd_general_and_origin_feats.sql` in `player-character-service/backend/src/main/resources/db/migration/`.
- [x] 1.2 Insert all missing SRD 2014 General Feats (40+ feats: Alerta, Atleta, Ator, Conjurador de Guerra, Especialista em Besta, Mestre de Armas Grandes, Mestre de Escudo, Mestre de Armas de Haste, Sentinela, Franco-Atirador, Sortudo, Robusto, etc.) with localized names, rules text, and prerequisites.
- [x] 1.3 Insert all missing SRD 2024 General Feats (30+ feats: Athlete, Charger, Crossbow Expert, Defensive Duelist, Dual Wielder, Great Weapon Master, Polearm Master, Sentinel, Sharpshooter, Skill Expert, etc.).
- [x] 1.4 Insert missing SRD 2024 Origin Feats (*Crafter, Healer, Lucky, Musician, Tavern Brawler*).
- [x] 1.5 Update legacy entries from V14 (*Iniciado em Magia / Magic Initiate*, *Atacante Selvagem / Savage Attacker*, *Habilidoso / Skilled*) to set `feat_type = 'ORIGIN'`.
- [x] 1.6 Populate structured `mechanical_effects` including `spellChoice` metadata for *Iniciado em Magia* and *Conjurador de Rituais* to support future inline accordion spellbook picker.

## Phase 2: Backend Persistence & Domain Support
- [x] 2.1 Update `MechanicalEffects.java` to include `spellChoice` record/metadata (`cantripsCount`, `spellsCount`, `allowedClasses`).
- [x] 2.2 Update `FeatJpaEntity.java` `deserializeEffects` to deserialize full `MechanicalEffects` using Jackson `ObjectMapper`, resolving the dropped prerequisites and ability bonus choices gap.
- [x] 2.3 Add unit tests in `MechanicalEffectsTest.java` verifying complete round-trip JSON serialization and deserialization.

## Phase 3: Character Creation & Level-Up Validation
- [x] 3.1 Verify `CreateCharacterService` prerequisite validator uses dynamic `prerequisiteRules`.
- [x] 3.2 Verify `LevelUpService` prerequisite validator evaluates minimum ability scores and character level for newly seeded feats.
- [x] 3.3 Add controller test in `ReferenceDataControllerTest.java` verifying `/api/reference/feats` returns the complete catalog under `featTypes=GENERAL` and `featTypes=ORIGIN`.

## Phase 4: Frontend Catalog & UI Verification
- [x] 4.1 Verify `/encyclopedia/feats` displays all newly seeded General and Origin feats with search and multi-select chips.
- [x] 4.2 Verify `/characters/new` background selection properly displays origin feats for all backgrounds.
- [x] 4.3 Verify Level Up Wizard rich feat selector loads and filters the expanded general feats catalog.

## Phase 5: Verification & Testing
- [x] 5.1 Run backend unit and integration tests (`mvn test -f player-character-service/backend/pom.xml`).
- [x] 5.2 Run frontend unit tests (`npm --prefix dungeoneer-frontend run test:unit`).
- [x] 5.3 Run frontend production build (`npm --prefix dungeoneer-frontend run build`).
- [x] 5.4 Rebuild Docker container `player-character-service` and verify Flyway V26 migration applies cleanly.

## Review Gate
- [x] All 6 acceptance criteria validated.
- [x] All 40+ SRD 2014 general feats available in encyclopedia and level-up.
- [x] All 10 SRD 2024 origin feats available with `feat_type = 'ORIGIN'`.
- [x] `spellChoice` metadata populated in `mechanical_effects` for Magic Initiate.
- [x] Prerequisite evaluation functional and covered by unit tests.
- [x] Conventional commit prepared for user approval.
