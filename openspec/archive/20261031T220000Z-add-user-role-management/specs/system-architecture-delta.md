# Delta Spec: `system-architecture`

> **Change ID:** `add-user-role-management`
> **Target Canonical Spec:** `openspec/specs/system-architecture.md`
> **Change Type:** Modify

## Architectural Updates

### User Registration & Multiple Roles
- O serviço de autenticação (`auth-service`) passa a aceitar um array opcional de `roles` no payload de registro (`POST /api/auth/register`). Caso omitido ou vazio, o fallback default é `[Role.PLAYER]`.
- Quando selecionados múltiplos módulos (Jogador e Mestre de Jogo), o usuário é persistido com ambas as roles (`PLAYER` e `DUNGEON_MASTER`), e o JWT gerado no login/registro reflete todos os papéis no claim `"roles"`.
- A data de nascimento suporta nativamente tanto o padrão brasileiro `dd/MM/yyyy` quanto o formato ISO `yyyy-MM-dd` via deserializador flexível.

### Self-Promotion & Token Reemission
- O endpoint `POST /api/auth/me/roles/promote-dm` permite a qualquer usuário autenticado auto-promover-se a `DUNGEON_MASTER`.
- O endpoint atua estritamente sobre o subject do JWT autenticado e retorna um novo `AuthResponseDto` com token JWT reemitido e assinado contendo a role `DUNGEON_MASTER`.
- O `api-gateway` reconhece rotas `/api/auth/me/**` como rotas protegidas que exigem token Bearer e propagam os cabeçalhos de identidade verificada (`X-User-Id`, `X-User-Roles`).
