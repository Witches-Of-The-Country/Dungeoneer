# Proposal: Add Main Dashboard

> **Change ID:** `add-main-dashboard`
> **Status:** draft
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Dashboard principal pós-login exibindo os personagens do usuário e um menu de enciclopédia (Arquétipos, Magias, Talentos, Itens, Linhagens, Monstros e NPCs). Monstros e NPCs só aparecem para DM e exibem estado vazio 'Em breve'.

## Motivation

<!-- What problem does this solve?
     - Reference user pain points, bug reports, or product requirements.
     - Include links to issues, tickets, Slack threads, or ADRs if applicable. -->

## Scope

### In Scope
- Layout/navegação do dashboard; listagem de personagens; menu de enciclopédia com visibilidade por role (DM); estado vazio para Monstros/NPCs.

### Out of Scope
- Busca/filtros (add-encyclopedia-browse); criação homebrew (add-homebrew-creation); gestão de roles (add-user-role-management).

## Proposed Solution

### Approach
<!-- Describe the technical approach in 2-4 paragraphs.
     Cover: what will be created/modified/deleted, why this approach was chosen,
     and any significant trade-offs. -->

### Impacted Files
<!-- Populated by sdd-code-explorer subagent. -->
| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/app/characters/page.tsx` | Modify | Integrar menu de enciclopédia ao dashboard |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Links de enciclopédia, gating por DM |
| `dungeoneer-frontend/lib/hooks/use-auth.ts` | Modify | Expor isDM/isPlayer (DUNGEON_MASTER) |
| `dungeoneer-frontend/components/encyclopedia-menu.tsx` | Create | Menu de categorias |
| `dungeoneer-frontend/app/encyclopedia/monsters/page.tsx` | Create | Placeholder DM 'Em breve' (e NPCs) |
| `dungeoneer-frontend/components/login-form.tsx` | Modify | Verificar redirect |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Verificar redirect |
| `dungeoneer-frontend/__tests__/main-dashboard.test.tsx` | Create | Testes do dashboard |

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


