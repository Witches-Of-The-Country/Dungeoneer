# Proposal: Add User Role Management

> **Change ID:** `add-user-role-management`
> **Status:** draft
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Tela de gerenciamento de usuário onde qualquer usuário pode se auto-promover a DM (risco aceito). Corrige dois bugs de cadastro: (1) selecionar DM e Player persiste só Player; (2) data de nascimento deve usar o formato dd/mm/yyyy (pt-BR).

## Motivation

<!-- What problem does this solve?
     - Reference user pain points, bug reports, or product requirements.
     - Include links to issues, tickets, Slack threads, or ADRs if applicable. -->

## Scope

### In Scope
- Tela de perfil/roles; endpoint de atualização de roles do próprio usuário; correção da persistência de múltiplas roles no cadastro; correção do formato/parse da data de nascimento.

### Out of Scope
- Role ADMIN; aprovação por terceiros; campanhas.

## Proposed Solution

### Approach
<!-- Describe the technical approach in 2-4 paragraphs.
     Cover: what will be created/modified/deleted, why this approach was chosen,
     and any significant trade-offs. -->

### Impacted Files
<!-- Populated by sdd-code-explorer subagent. -->
| File | Change Type | Notes |
|------|-------------|-------|
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java` | Modify | createNew aceitar roles; addRole |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/dto/RegisterRequest.java` | Modify | Campo roles; birthdate dd/MM/yyyy |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/RegisterUserCommand.java, application/service/AuthService.java` | Modify | Propagar roles; auto-promoção |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/SelfPromoteUseCase.java` | Create | Porta de auto-promoção |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/out/persistence/UserPersistenceAdapter.java` | Modify | Corrigir persist vs merge em updates (isNew) |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java` | Modify | register com roles; endpoint self-promote com novo JWT |
| `api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java` | Modify | Proteger /api/auth/me/** |
| `dungeoneer-frontend/lib/api/auth.ts` | Modify | roles no payload; selfPromoteToDm() |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Mapear modules->roles; máscara dd/mm/yyyy |
| `dungeoneer-frontend/app/profile/page.tsx` | Create | Tela de perfil/papéis |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Link de perfil |

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


