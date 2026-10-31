# Proposal: Add Main Dashboard

> **Change ID:** `add-main-dashboard`
> **Status:** in-progress
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Dashboard principal pós-login unificado em `/characters`, apresentando os personagens do jogador e um menu navegável de enciclopédia (Arquétipos, Magias, Talentos, Itens, Linhagens, Monstros e NPCs). O acesso e a visibilidade dos menus de Monstros e NPCs são restritos a usuários com o papel `DUNGEON_MASTER` (Mestre), exibindo páginas de estado vazio ("Em breve") no tema Dark Fantasy.

## Motivation

Atualmente, a tela pós-login foca exclusivamente na listagem de personagens (`/characters`). Os jogadores e mestres precisam de uma central integrada para navegar pelo compêndio de regras, itens e magias, enquanto mestres necessitam de ferramentas dedicadas para preparação de campanhas (monstros e NPCs). Essa mudança unifica o hub inicial da aplicação e estabelece a base para a navegação de enciclopédia e ferramentas de mestre.

## Scope

### In Scope
- Layout integrado do Dashboard principal em `dungeoneer-frontend/app/characters/page.tsx` com seção de personagens e seção de Enciclopédia & Referências.
- Componente `EncyclopediaMenu` exibindo cards com ícones e rótulos em pt-BR (Arquétipos, Magias, Talentos, Itens, Linhagens, Monstros e NPCs).
- Helpers no hook de autenticação `useAuth` (`isDM`, `isPlayer`) para checagem reativa de papéis (`DUNGEON_MASTER`, `PLAYER`).
- Gating condicional dos cards de Monstros e NPCs para usuários com papel `DUNGEON_MASTER`, incluindo badge "Mestre".
- Rotas de placeholder `/encyclopedia/monsters` e `/encyclopedia/npcs` com estado vazio Dark Fantasy ("Em breve"), botão de retorno e proteção de rota para não-DMs.
- Atualização da `Navbar` com links rápidos e badge de "Mestre".
- Verificação de redirecionamento pós-login e pós-registro para `/characters`.
- Suíte completa de testes unitários automatizados (Vitest + Testing Library).

### Out of Scope
- Busca com carregamento lazy e filtros avançados (escopo de `add-encyclopedia-browse`).
- Botão "Criar novo" para conteúdo homebrew (escopo de `add-homebrew-creation`).
- Gestão de roles e auto-promoção a DM (escopo de `add-user-role-management`).
- Banco de dados e endpoints de monstros/NPCs no backend.

## Proposed Solution

### Approach
1. **Extensão de Autenticação (`use-auth.ts`)**:
   - Adicionar propriedades booleanas `isDM` e `isPlayer` ao hook `useAuth`, derivadas de `user?.roles?.includes('DUNGEON_MASTER')` e `user?.roles?.includes('PLAYER')`, com fallback seguro para `false`.
2. **Componente de Menu de Enciclopédia (`encyclopedia-menu.tsx`)**:
   - Construir um componente reutilizável em Next.js / TailwindCSS 4 com estilo Dark Fantasy RPG (bordas sutis `border-zinc-800`, hover dourado `border-amber-500/50`, ícones Lucide).
   - Renderizar as 5 categorias base para todos os usuários autenticados.
   - Renderizar Monstros e NPCs somente se `isDM` for verdadeiro, destacando badge estilizada de "Mestre".
3. **Páginas Placeholder com Route Guard (`/encyclopedia/monsters` e `/encyclopedia/npcs`)**:
   - Renderizar aviso de acesso restrito caso um usuário sem a role `DUNGEON_MASTER` acesse diretamente a URL.
   - Apresentar card temático com ícone, título "Em breve" e texto descritivo para DMs.
4. **Integração no Dashboard e Navbar**:
   - Integrar o `EncyclopediaMenu` na página `/characters` abaixo da listagem de personagens.
   - Adicionar atalho na barra de navegação superior (`navbar.tsx`).

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/app/characters/page.tsx` | Modify | Integrar seção de enciclopédia ao dashboard de personagens |
| `dungeoneer-frontend/components/navbar.tsx` | Modify | Adicionar links e badge de DM na barra de navegação |
| `dungeoneer-frontend/lib/hooks/use-auth.ts` | Modify | Expor flags `isDM` e `isPlayer` |
| `dungeoneer-frontend/components/encyclopedia-menu.tsx` | Create | Componente de cards de navegação da enciclopédia |
| `dungeoneer-frontend/app/encyclopedia/monsters/page.tsx` | Create | Página placeholder de monstros com guard de DM |
| `dungeoneer-frontend/app/encyclopedia/npcs/page.tsx` | Create | Página placeholder de NPCs com guard de DM |
| `dungeoneer-frontend/components/login-form.tsx` | Modify | Verificar/assegurar redirect pós-login para `/characters` |
| `dungeoneer-frontend/components/register-form.tsx` | Modify | Verificar/assegurar redirect pós-registro para `/characters` |
| `dungeoneer-frontend/__tests__/use-auth-roles.test.ts` | Create | Testes unitários para as flags de role no useAuth |
| `dungeoneer-frontend/__tests__/encyclopedia-menu.test.tsx` | Create | Testes unitários do EncyclopediaMenu e role-gating |
| `dungeoneer-frontend/__tests__/encyclopedia-placeholders.test.tsx` | Create | Testes unitários das páginas placeholder e route guards |
| `dungeoneer-frontend/__tests__/main-dashboard.test.tsx` | Create | Testes de integração do dashboard unificado |

### Data Model Changes (if applicable)
N/A - Esta mudança é restrita ao frontend e utiliza as roles já persistidas no JWT (`PLAYER`, `DUNGEON_MASTER`).

## Acceptance Criteria

- [ ] **Given** um usuário logado com papel `PLAYER`, **When** acessar a página `/characters`, **Then** deve visualizar a seção "Meus Personagens" e a seção "Enciclopédia & Referências" contendo apenas Arquétipos, Magias, Talentos, Itens e Linhagens, sem exibir Monstros e NPCs.
- [ ] **Given** um usuário logado com papel `DUNGEON_MASTER`, **When** acessar a página `/characters`, **Then** deve visualizar todos os cards de enciclopédia, incluindo Monstros e NPCs destacados com o badge "Mestre".
- [ ] **Given** um usuário logado com papel `DUNGEON_MASTER`, **When** navegar para `/encyclopedia/monsters` ou `/encyclopedia/npcs`, **Then** deve visualizar a tela com título "Em breve" e opção de retornar ao dashboard.
- [ ] **Given** um usuário logado apenas com papel `PLAYER`, **When** tentar acessar diretamente a URL `/encyclopedia/monsters` ou `/encyclopedia/npcs`, **Then** deve ser exibida uma mensagem de "Acesso restrito ao Mestre do Jogo" com botão para voltar ao dashboard.
- [ ] **Given** um usuário realizando login ou cadastro com sucesso, **When** a autenticação for concluída, **Then** o sistema deve redirecioná-lo para `/characters`.

## Defensive Security & Validation Considerations

### Input Validation
- Validação no cliente das URLs e parâmetros de rota.
- Tratamento resiliente caso a lista de roles do usuário venha nula, indefinida ou em formato inesperado.

### Secret Isolation
- Nenhuma chave secreta ou credencial é exposta no frontend.
- O token JWT existente em cookies/localStorage é mantido com os mesmos padrões de segurança sem alteração de exposição.

### Authorization & Authentication Impact
- Client-side role gating: Embora a segurança em profundidade exija proteção nos endpoints do backend (a ser reforçada nas mudanças subsequentes do gateway), a interface e as rotas Next.js bloqueiam acessos indevidos de jogadores a ferramentas exclusivas de DM.

### Other Security Risks
- Spoofing de role no cliente: Usuários podem manipular `localStorage` no browser, porém as telas de monstros e NPCs são meros placeholders sem dados sensíveis de backend nessa fase. Endpoints reais futuros serão protegidos por Gateway/JWT.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Helper `useAuth` com role `DUNGEON_MASTER` | Unit | `isDM` é `true`, `isPlayer` é `false` (ou `true` se possuir ambas) |
| TP-02 | Helper `useAuth` com role `PLAYER` | Unit | `isDM` é `false`, `isPlayer` é `true` |
| TP-03 | Renderização do `EncyclopediaMenu` para jogador comum | Unit | Renderiza 5 categorias base e oculta Monstros e NPCs |
| TP-04 | Renderização do `EncyclopediaMenu` para DM | Unit | Renderiza 7 categorias incluindo Monstros e NPCs com badge |
| TP-05 | Acesso a `/encyclopedia/monsters` como DM | Unit | Exibe card "Em breve" e botão de retorno |
| TP-06 | Dashboard principal `/characters` integrado | Integration | Exibe lista de personagens e grid de enciclopédia |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Usuário não autenticado ou roles vazias no `useAuth` | Unit | `isDM` e `isPlayer` retornam `false` com segurança sem erro de runtime |
| TE-02 | Acesso direto de jogador comum a `/encyclopedia/monsters` ou `/npcs` | Unit | Exibe aviso de acesso restrito ao Mestre |

### Test Commands
- **Run tests:** `npm --prefix dungeoneer-frontend run test:unit`
- **Build check:** `npm --prefix dungeoneer-frontend run build`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Inconsistência de layout entre grid de personagens e enciclopédia | Baixa | Média | Uso de componentes responsivos consistentes com tokens Dark Fantasy (`bg-zinc-900/60`, `border-zinc-800`, `text-zinc-100`) |
| Tentativa de acesso manual via URL por jogadores normais | Média | Baixa | Fallback de Route Guard nas páginas placeholder avisando restrição de Mestre |

## Specification Changes & Delta Stubs

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Adiciona a especificação do Dashboard unificado pós-login, menu de Enciclopédia (`EncyclopediaMenu`), checagem reativa de papéis no cliente (`isDM`, `isPlayer`), rotas de placeholder (`/encyclopedia/monsters`, `/encyclopedia/npcs`) e padrão visual de estado vazio Dark Fantasy.

## References
- `openspec/specs/frontend-web-app.md`
- `openspec/specs/system-architecture.md`
- `prompt-01-add-main-dashboard.md`
