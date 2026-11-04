# Proposal: Seed Complete SRD General and Origin Feats

> **Change ID:** `009-seed-complete-srd-general-and-origin-feats`
> **Status:** draft
> **Author:** Antigravity & User Pair
> **Created:** 2026-11-02
> **Last Updated:** 2026-11-02

---

## Summary

Seed the complete collection of official System Reference Document (SRD) General and Origin Feats for both **SRD 2014** and **SRD 2024** rulesets via Flyway migration `V26__seed_complete_srd_general_and_origin_feats.sql`. This fills the current database gap where SRD 2014 contains only a single general feat (`grappler-2014`) and SRD 2024 lacks critical general feats (such as *Great Weapon Master, Sentinel, Sharpshooter, Polearm Master, Skill Expert*) and origin feats (such as *Crafter, Healer, Lucky, Musician, Tavern Brawler*). It also fixes `feat_type = 'ORIGIN'` on existing 2024 origin feats, upgrades `FeatJpaEntity.deserializeEffects` to deserialize full mechanical effects (including prerequisites, ability bonus choices, and future spell-granting metadata for feats like *Magic Initiate*), and validates prerequisites in character creation and level-up.

## Motivation

Players building characters in the Character Creation Wizard (`/characters/new`) or leveling up in the Level Up Wizard (`/characters/[id]`) currently encounter a severely limited list of feats:
1. SRD 2014 has over 40 official general feats, but the database only has 1 (`grappler-2014`).
2. SRD 2024 lacks iconic combat and utility feats (*Mestre de Armas Grandes, Sentinela, Franco-Atirador, Especialista em Perícia, etc.*).
3. Several SRD 2024 origin feats (*Iniciado em Magia, Atacante Selvagem, Habilidoso*) are incorrectly categorized as `GENERAL` instead of `ORIGIN`.
4. Feats like *Magic Initiate* (Iniciado em Magia) will need metadata specifying granted cantrips and spells so that when the Spellbook / Grimório system is implemented, the UI can seamlessly render an inline accordion to select spells without disruptive nested modal popups.
5. In the backend, `FeatJpaEntity.deserializeEffects` drops `prerequisiteRules` and `abilityBonusChoice` during JPA conversion, causing prerequisite verification to fall back to hardcoded strings rather than database-driven rules.

## Scope

### In Scope
- **Flyway Migration `V26`**:
  - Insert all missing SRD 2014 General Feats (40+ feats including *Alerta, Atleta, Conjurador de Guerra, Especialista em Besta, Mestre de Armas Grandes, Mestre de Escudo, Mestre de Armas de Haste, Sentinela, Franco-Atirador, Sortudo, Robusto, etc.*).
  - Insert all missing SRD 2024 General Feats (30+ feats including *Athlete, Charger, Crossbow Expert, Defensive Duelist, Dual Wielder, Great Weapon Master, Polearm Master, Sentinel, Sharpshooter, Skill Expert, etc.*).
  - Insert missing SRD 2024 Origin Feats (*Crafter, Healer, Lucky, Musician, Tavern Brawler*).
  - Normalize existing SRD 2024 entries (*Magic Initiate, Savage Attacker, Skilled*) with `feat_type = 'ORIGIN'`.
  - Include rich `mechanical_effects` JSON payloads with ability bonuses, prerequisite rules, and future spell-choice metadata (`"spellChoice": { "cantripsCount": 2, "spellsCount": 1, "allowedClasses": ["CLERIC", "DRUID", "WIZARD"], "spellLevel": 1 }`).
- **Backend Deserialization Fix**:
  - Update [FeatJpaEntity.java](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntity.java) `deserializeEffects` to parse `abilityBonusChoice`, `conditional`, `conditionDescription`, and `prerequisiteRules`.
- **Character Creation & Level-Up Validation**:
  - Ensure [CreateCharacterService.java](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java) and [LevelUpService.java](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java) properly evaluate DB-stored prerequisite rules.
- **Frontend Verification**:
  - Verify character wizard background origin feat selection matches the expanded origin feats list.
  - Verify Level Up Wizard rich feat selector displays all general feats with search, pagination, and details modal.
  - Verify `/encyclopedia/feats` lists all newly seeded feats with filter chips (`Origem`, `Geral`, `Todos`).

### Out of Scope
- Full interactive spellbook modal implementation (will consume the `spellChoice` metadata in a dedicated Grimório spec).
- Homebrew feat creation wizard enhancements (already implemented in Spec 002).
- Non-SRD third-party or homebrew published content.

## Proposed Solution

### Approach
1. **Migration `V26__seed_complete_srd_general_and_origin_feats.sql`**:
   - Assign deterministic UUIDs in the `30000000-0000-0000-0000-000000000700` range for 2014 and `40000000-0000-0000-0000-000000000700` range for 2024.
   - Use `ON CONFLICT (ruleset, slug) DO UPDATE` to ensure idempotency and correct existing entries without affecting foreign keys.
   - Localize all names (`name` in Portuguese, `name_en` in English), provide clear rules text, human-readable `prerequisites`, and structured `mechanical_effects`.
   - For feats that grant spell choices (*Iniciado em Magia / Magic Initiate, Conjurador de Rituais / Ritual Caster*), populate `mechanical_effects.spellChoice` to support future spellbook picker integration seamlessly.
2. **Backend JPA Entity Mapping**:
   - Enhance `FeatJpaEntity.deserializeEffects` to parse all properties of `MechanicalEffects` using Jackson `ObjectMapper`, eliminating manual partial instantiation.
3. **Automated Testing**:
   - Write integration and unit tests verifying that all seeded feats are retrieved via `/api/reference/feats`, that filtering by `featTypes=GENERAL` and `featTypes=ORIGIN` returns complete catalogs, and that prerequisite rules are honored.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V26__seed_complete_srd_general_and_origin_feats.sql` | Create | Flyway migration seeding all SRD 2014 and 2024 General & Origin feats |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntity.java` | Modify | Update `deserializeEffects` to full Jackson deserialization |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffectsTest.java` | Modify | Add tests for full deserialization including spell choices |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataControllerTest.java` | Modify | Verify full feat catalog retrieval and filtering |
| `dungeoneer-frontend/__tests__/level-up-wizard.test.tsx` | Modify | Verify Level Up Wizard loads full general feats catalog |

### Data Model Changes
No new tables or altered column definitions are required. The existing `player_character.feats` table and `mechanical_effects` JSONB column natively support all proposed payloads.

## Acceptance Criteria

- [ ] **Given** a user viewing `/encyclopedia/feats` with filter `Geral` and `SRD_2014`, **When** the page loads, **Then** all 40+ official SRD 2014 general feats (*Alerta, Atleta, Conjurador de Guerra, Mestre de Armas Grandes, etc.*) are listed with pagination.
- [ ] **Given** a user viewing `/encyclopedia/feats` with filter `Origem` and `SRD_2024`, **When** the page loads, **Then** all 10 SRD 2024 origin feats (*Alerta, Vigoroso, Iniciado em Magia, Atacante Selvagem, Habilidoso, Artesão, Curandeiro, Sortudo, Músico, Brigão de Taverna*) are displayed.
- [ ] **Given** a player creating a 2024 character in `/characters/new`, **When** selecting any background, **Then** the corresponding Origin Feat is found and displayed correctly.
- [ ] **Given** a character leveling up to an ASI level in `/characters/[id]`, **When** choosing the "Talento" option in the Level Up Wizard, **Then** the rich feat selector displays the full catalog of general feats with search and details.
- [ ] **Given** the *Iniciado em Magia* (Magic Initiate) feat, **When** its `mechanical_effects` JSON is inspected, **Then** it contains structured `spellChoice` metadata (`cantripsCount`, `spellsCount`, `allowedClass`) ready for inline accordion selection in the character sheet / spellbook.
- [ ] **Given** a feat with ability score prerequisites (e.g. Força 13+), **When** a character lacking the prerequisite attempts to select it, **Then** the backend rejects the selection with a 400 Bad Request ProblemDetail.

## Defensive Security & Validation Considerations

### Input Validation
- All feat selections during character creation and level-up are validated against character ability scores, level, and declared ruleset in [CreateCharacterService.java](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java) and [LevelUpService.java](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java).
- SQL injection is prevented by Flyway static migration scripts and parameterized JPA queries.

### Secret Isolation
- No secrets or credentials are introduced. Database access remains entirely environment-variable-driven.

### Authorization & Authentication Impact
- Public reference endpoints (`/api/reference/feats`) remain accessible to players and guests for encyclopedia and character building.
- Homebrew creation endpoints continue to require DM role (`ROLE_DUNGEON_MASTER`).

### Other Security Risks
- All JSONB payloads are deserialized with bounded schema types to prevent remote code execution or arbitrary class instantiation.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Flyway migration V26 execution | Integration | All migrations apply cleanly from V1 to V26 in Testcontainers |
| TP-02 | GET /api/reference/feats with featTypes=GENERAL | API Unit | Returns 200 OK with paginated list of general feats |
| TP-03 | GET /api/reference/feats with featTypes=ORIGIN | API Unit | Returns 200 OK with all 10 SRD 2024 origin feats |
| TP-04 | Deserialization of `mechanical_effects` in FeatJpaEntity | Unit | Parses ability bonuses, choices, prerequisites, and spellChoice |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Selecting a feat without meeting minimum ability score | Unit / Integration | 400 Bad Request with ProblemDetail stating missing prerequisite |
| TE-02 | Cross-ruleset feat query (SRD_2014 vs SRD_2024) | API Unit | Query strictly respects ruleset parameter and never leaks cross-ruleset feats |

### Test Commands
- **Backend tests:** `mvn test -f player-character-service/backend/pom.xml`
- **Frontend tests:** `npm --prefix dungeoneer-frontend run test:unit`
- **Frontend build:** `npm --prefix dungeoneer-frontend run build`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Slug collisions with existing V14/V16 feats | Medium | Medium | Use `ON CONFLICT (ruleset, slug) DO UPDATE` in `V26` to overwrite/enhance rather than duplicate |
| Typo in prerequisite label breaks UI display | Low | Low | Standardized naming conventions validated via automated entity deserialization unit test |
