# Proposal: Add User Role Management

> **Change ID:** `add-user-role-management`
> **Status:** draft
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Tela de gerenciamento de usuário onde qualquer usuário pode se auto-promover a Dungeon Master (`DUNGEON_MASTER`), além de permitir gerenciar papéis de usuário. Corrige simultaneamente dois bugs críticos no fluxo de cadastro: (1) no formulário de registro, selecionar DM e Player persistia exclusivamente a role `PLAYER`; (2) a data de nascimento no frontend e backend deve suportar o formato padrão brasileiro `dd/MM/yyyy` com validação estrita.

---

## Motivation

Atualmente, usuários que se cadastram marcando os módulos de Mestre e Jogador acabam recebendo unicamente a role `PLAYER` devido a um hardcode no factory method do agregado `User` (`List.of(Role.PLAYER)`). Isso impede que esses usuários utilizem funcionalidades exclusivas de Mestre, como a criação e gerenciamento de conteúdo Homebrew e campanhas. Além disso, o formato de data de nascimento no formulário de cadastro causa atritos de validação e experiência de usuário para o público brasileiro. Permitir a auto-promoção a `DUNGEON_MASTER` na tela de perfil do usuário e aceitar múltiplas roles no registro resolve completamente os impedimentos de acesso às ferramentas de DM.

---

## Scope

### In Scope
- Endpoint de auto-promoção e atualização de roles para o próprio usuário autenticado (`POST /api/auth/me/roles` ou `/api/auth/users/{userId}/roles`), reemitindo novo token JWT contendo os papéis atualizados.
- Ajuste no formulário de cadastro (`RegisterRequest` e formulário no frontend) para propagar a lista de roles selecionadas (ex: `[PLAYER, DUNGEON_MASTER]`).
- Suporte a parsing e validação de data de nascimento no formato `dd/MM/yyyy` (pt-BR) além do padrão ISO `yyyy-MM-dd`.
- Tela de perfil do usuário (`/profile`) permitindo visualizar as informações cadastrais e ativar a role de `DUNGEON_MASTER`.
- Link de navegação para a página de perfil no `Navbar`.
- Proteção e propagação no `api-gateway` para rotas `/api/auth/me/**`.

### Out of Scope
- Role administrativa `ADMIN` ou painel central de moderação/gestão de todos os usuários.
- Fluxo de aprovação ou convite por terceiros para liberação de permissão de Mestre.
- Gestão de permissões de campanhas específicas ou permissões granulares por sala.

---

## Proposed Solution

### Approach
No backend (`auth-service`), expandiremos o `RegisterRequest`, o `RegisterUserCommand` e o método `User.createNew` para aceitar uma lista opcional de roles. Caso nenhuma role seja informada no cadastro, o valor padrão continuará sendo `[Role.PLAYER]`. Criaremos a porta de entrada `SelfPromoteUseCase` (ou estenderemos `UpdateUserRolesUseCase`), permitindo que um usuário autenticado solicite a adição da role `DUNGEON_MASTER` ao seu próprio perfil. A persistência em `UserPersistenceAdapter` será validada para assegurar que chamadas de atualização executem o salvamento e merge corretos das collections de roles. Ao atualizar as roles do usuário, o backend reemitirá um novo token JWT assinado contendo as claims de roles atualizadas (`"roles": ["PLAYER", "DUNGEON_MASTER"]`).

Para a data de nascimento, configuraremos o deserializador de `RegisterRequest.birthdate` para aceitar `dd/MM/yyyy` (usando `@JsonFormat(pattern = "dd/MM/yyyy")` ou suporte flexível via fallback), acompanhado de máscara de entrada e formatação correspondente no componente `RegisterForm` do frontend.

No frontend (`dungeoneer-frontend`), criaremos a página de perfil em `/profile`, onde o usuário pode visualizar seu nickname, email, data de nascimento e seus papéis ativos. Haverá um botão de ação "Tornar-se Mestre de Jogo (DM)" que invocará o endpoint de auto-promoção, salvará o novo token JWT e o perfil atualizado no storage local e disparará o evento `auth-state-change` para atualizar instantaneamente o Navbar e as permissões de interface.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `auth-service/backend/src/main/java/com/dungeoneer/auth/domain/model/User.java` | Modify | `createNew` passa a aceitar coleção de roles com fallback para `PLAYER` |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/dto/RegisterRequest.java` | Modify | Adicionar campo `List<Role> roles` e anotação `@JsonFormat(pattern = "dd/MM/yyyy")` ou parser flexível |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/port/in/RegisterUserCommand.java` | Modify | Adicionar `List<Role> roles` ao record de comando |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/application/service/AuthService.java` | Modify | Propagar roles no registro e implementar auto-promoção com novo JWT |
| `auth-service/backend/src/main/java/com/dungeoneer/auth/adapter/in/web/controller/AuthController.java` | Modify | Endpoints de registro e auto-promoção com reemissão de JWT |
| `api-gateway/backend/src/main/java/com/dungeoneer/api_gateway/security/JwtHeaderFilter.java` | Modify | Garantir propagação correta para `/api/auth/me/**` |
| `dungeoneer-frontend/lib/api/auth.ts` | Modify | Suporte a roles no `RegisterData` e função `selfPromoteToDm()` |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Mapeamento dos checkboxes de módulos para roles e máscara `dd/MM/yyyy` |
| `dungeoneer-frontend/app/profile/page.tsx` | Create | Tela de perfil e auto-promoção de papéis |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Adicionar link de navegação para a página de perfil |

---

## Acceptance Criteria

- [ ] **Given** um novo usuário realizando cadastro no frontend marcando os módulos de Jogador e Mestre **When** o formulário enviar o payload com `roles: ["PLAYER", "DUNGEON_MASTER"]` **Then** o usuário é persistido com ambas as roles e o JWT gerado contém `PLAYER` e `DUNGEON_MASTER`.
- [ ] **Given** um usuário preenchendo a data de nascimento no cadastro **When** a data for digitada no formato `dd/MM/yyyy` (ex: `25/12/1995`) **Then** o backend interpreta corretamente o dia, mês e ano no fuso local sem erros de validação.
- [ ] **Given** um usuário autenticado que possui unicamente o papel `PLAYER` **When** o usuário acessar a tela `/profile` e clicar em "Tornar-se Mestre de Jogo (DM)" **Then** o sistema adiciona a role `DUNGEON_MASTER`, reemite um novo token JWT com a role atualizada e atualiza o estado da aplicação sem necessidade de relogar.

---

## Defensive Security & Validation Considerations

### Input Validation
- O payload de registro (`RegisterRequest`) e de atualização de roles utiliza Jakarta Validation (`@NotNull`, `@NotEmpty`, `@Size`).
- As roles enviadas devem pertencer aos valores válidos do enum `Role` (`PLAYER`, `DUNGEON_MASTER`).
- A data de nascimento deve ser estritamente no passado (`@Past` ou validação de domínio em `User`).

### Secret Isolation
- Nenhuma chave secreta ou credencial é exposta. A assinatura dos tokens JWT utiliza o `JWT_SECRET` injetado via variável de ambiente.

### Authorization & Authentication Impact
- A rota `/api/auth/me/**` exige token JWT válido com cabeçalho `Authorization: Bearer <token>`.
- O endpoint de auto-promoção atua estritamente sobre o `userId` autenticado extraído do token JWT verificado pelo gateway, prevenindo que um usuário altere as roles de outro usuário.

### Other Security Risks
- Risco aceito de auto-promoção aberta para `DUNGEON_MASTER` (adequado para o escopo do MVP de RPG de mesa cooperativo sem necessidade de aprovação externa).

---

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Cadastro com roles `[PLAYER, DUNGEON_MASTER]` | Integration (MockMvc) | Retorna 201 Created com JWT contendo ambas as roles |
| TP-02 | Cadastro com data de nascimento `dd/MM/yyyy` | Integration (MockMvc) | Retorna 201 Created e salva a data correta no banco |
| TP-03 | Auto-promoção do próprio usuário para `DUNGEON_MASTER` | Integration (MockMvc) | Retorna 200 OK com novo JWT refletindo a role |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Cadastro com data de nascimento no futuro | Unit / Integration | Retorna 422 Unprocessable Entity |
| TE-02 | Auto-promoção sem token ou com token inválido | Integration | Retorna 401 Unauthorized |
| TE-03 | Envio de role nula ou inválida | Integration | Retorna 422 Unprocessable Entity |

### Test Commands
- **Backend tests:** `mvn clean test -f auth-service/backend/pom.xml`
- **Frontend tests:** `npm run test:unit --prefix dungeoneer-frontend`

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Inconsistência de token no navegador após auto-promoção | Média | Alta | Salvar imediatamente o novo `accessToken` retornado e disparar o evento `auth-state-change`. |
| Erro de timezone ou formato ao deserializar `dd/MM/yyyy` | Média | Média | Adicionar testes com datas limites e usar `@JsonFormat(pattern = "dd/MM/yyyy")`. |
| Usuário perder a role `PLAYER` ao se auto-promover | Baixa | Média | Garantir no domínio que `addRole` adiciona à lista sem sobrescrever as existentes. |

---

## Specification Changes & Delta Stubs

### `specs/system-architecture-delta.md`
- **Target Canonical Spec:** `openspec/specs/system-architecture.md`
- **Summary of Updates:** Adição das regras de propagação de rotas `/api/auth/me/**` e reemissão de tokens JWT em atualizações de papéis de usuário.

---

## References

- [sdd-stack-contract.md](file:///D:/Projects/Dungeoneer/openspec/specs/sdd-stack-contract.md)
- [system-architecture.md](file:///D:/Projects/Dungeoneer/openspec/specs/system-architecture.md)
