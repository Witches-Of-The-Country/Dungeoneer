# Proposal: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`
> **Status:** draft
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Telas de consulta para Arquétipos, Magias, Talentos, Itens e Linhagens com carregamento lazy (infinite scroll por cursor), filtros e pesquisa por nome e texto da descrição via PostgreSQL full-text search.

## Motivation

<!-- What problem does this solve?
     - Reference user pain points, bug reports, or product requirements.
     - Include links to issues, tickets, Slack threads, or ADRs if applicable. -->

## Scope

### In Scope
- Endpoints paginados por cursor com filtros e parâmetro de busca; índices tsvector/GIN via Flyway; UI com infinite scroll (TanStack Query) e filtros.

### Out of Scope
- Criação de conteúdo; Monstros/NPCs; dashboard shell.

## Proposed Solution

### Approach
<!-- Describe the technical approach in 2-4 paragraphs.
     Cover: what will be created/modified/deleted, why this approach was chosen,
     and any significant trade-offs. -->

### Impacted Files
<!-- Populated by sdd-code-explorer subagent. -->
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V7__create_spells_feats_items_and_fts_indexes.sql` | Create | Tabelas spells/feats/items + índices GIN tsvector |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Item.java` | Create | Entidade de domínio ausente |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ (Spell/Feat/Item JpaEntity + SpringData repos)` | Create | Persistência e queries FTS/cursor |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataArchetypeRepository.java, SpringDataLineageRepository.java` | Modify | Cursor + FTS |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java, adapter/out/persistence/ReferenceDataPersistenceAdapter.java` | Modify | Consultas paginadas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/GetReferenceDataUseCase.java, application/service/ReferenceDataService.java` | Modify | Casos de uso paginados |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/ (CursorPageResponse, SpellDto, FeatDto, ItemDto)` | Create | DTOs |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java` | Modify | Parâmetros cursor/q/filtros; novos GET spells/feats/items |
| `dungeoneer-frontend/lib/api/encyclopedia.ts` | Create | useInfiniteQuery |
| `dungeoneer-frontend/app/encyclopedia/ (layout, classes, spells, feats, items, lineages)` | Create | Telas com infinite scroll, busca e filtros |

### Data Model Changes (if applicable)
<!-- Describe any changes to database schemas, API contracts, or shared types. -->

## Acceptance Criteria

<!-- Each criterion MUST follow Given-When-Then format and be independently verifiable. -->
- [ ] **Given** <!-- context --> **When** <!-- action --> **Then** <!-- expected result -->
- [ ] <!-- Criterion 2 -->
- [ ] <!-- Criterion 3 -->

## Defensive Security & Validation Considerations

<!-- MANDATORY SECTION â€” must be filled before proposal can be approved. -->

### Input Validation
- <!-- Describe what input data this change handles and how it will be validated.
     Example: "All incoming DTOs validated with @IsNotEmpty, @IsEmail Jakarta annotations." -->

### Secret Isolation
- <!-- Confirm no credentials, tokens, or keys are hardcoded.
     Example: "Database URL injected via DATABASE_URL env variable only." -->

### Authorization & Authentication Impact
- <!-- Describe if this change affects access control.
     Example: "New endpoint requires JWT with role=ADMIN. Enforced by AuthGuard." -->
     If not applicable, state explicitly: "N/A â€” this change does not expose endpoints or modify auth logic."

### Other Security Risks
- <!-- List any additional risks: SQL injection vectors, SSRF, path traversal, etc. -->

## Testing Strategy & Shift-Left Plan

<!-- MANDATORY SECTION â€” must be filled before proposal can be approved. -->

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | <!-- Normal usage description --> | Unit / Integration / E2E | <!-- Expected outcome --> |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | <!-- Boundary value, null input, timeout, etc. --> | Unit / Integration | <!-- Expected outcome --> |
| TE-02 | <!-- Unauthorized access attempt --> | Integration | <!-- Expected outcome --> |

### Test Commands
- **Run tests:** `<!-- e.g. npm test, pytest, ./gradlew test -->`
- **Coverage check:** `<!-- e.g. npm run test:coverage -->`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| <!-- Describe risk --> | Low / Med / High | Low / Med / High | <!-- How to mitigate --> |

## Specification Changes & Delta Stubs

<!-- Map code changes to affected architectural specifications.
     - For existing canonical specs: specs/<target>-delta.md
     - For new domain modules: specs/<new-module>.md (canonical template) -->

### `specs/<spec-name>-delta.md` (or `specs/<new-module>.md`)
- **Target Canonical Spec:** `openspec/specs/<target>.md` (or `New Specification`)
- **Summary of Updates:** <!-- Summary of functional/architectural changes to be consolidated upon archive -->

## References

- <!-- Link to related spec, ADR, or documentation -->
- <!-- Link to issue tracker ticket -->


