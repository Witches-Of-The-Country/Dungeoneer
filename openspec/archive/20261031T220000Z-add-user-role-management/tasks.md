# Tasks: Add User Role Management

> **Change ID:** `add-user-role-management`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] <!-- task: auth-user-roles-domain --> Atualizar `User.createNew` e construtores de `User` para suportar lista de roles configurável no registro.
- [x] <!-- task: auth-register-dto --> Atualizar `RegisterRequest` e `RegisterUserCommand` para incluir `roles` e configurar suporte a `dd/MM/yyyy` para `birthdate`.
- [x] <!-- task: auth-self-promote-endpoint --> Implementar endpoint de auto-promoção / gerenciamento de papéis (`/api/auth/me/roles/promote-dm`) que reemite o token JWT com a role `DUNGEON_MASTER`.
- [x] <!-- task: gateway-route-propagation --> Verificar propagação de headers de autenticação no `api-gateway` para endpoints `/api/auth/me/**`.
- [x] <!-- task: frontend-register-form --> Atualizar `RegisterForm` no frontend para enviar as roles selecionadas e formatar a data de nascimento como `dd/MM/yyyy`.
- [x] <!-- task: frontend-profile-page --> Criar a página de perfil (`app/profile/page.tsx`) com interface para auto-promoção a DM e feedback amigável.
- [x] <!-- task: frontend-navbar-link --> Adicionar link para a página de perfil no `Navbar`.

## Test Tasks

- [x] <!-- task: test-auth-register-multiple-roles --> Teste unitário e de integração no `auth-service` para registro com múltiplas roles (`PLAYER` e `DUNGEON_MASTER`).
- [x] <!-- task: test-auth-birthdate-format --> Teste de integração validando o parsing de data no formato `dd/MM/yyyy`.
- [x] <!-- task: test-auth-self-promote --> Teste de integração para auto-promoção a `DUNGEON_MASTER` com validação de emissão de novo JWT.
- [x] <!-- task: test-frontend-profile --> Testes unitários no frontend para a tela de perfil e formulário de cadastro.

## Documentation Tasks

- [x] Atualizar delta spec `specs/system-architecture-delta.md` com os fluxos de auto-promoção e renovação de token.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
