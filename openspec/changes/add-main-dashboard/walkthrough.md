# Walkthrough: Add Main Dashboard

> **Change ID:** `add-main-dashboard`  
> **Status:** Em andamento  
> **Data:** 31/10/2026  

---

## Fase 1: Finalização da Especificação e Gates SDD

Nesta fase inicial, completamos todos os documentos de especificação dentro de `openspec/changes/add-main-dashboard/`:
- `proposal.md`: Detalhamento dos objetivos, escopo, proposta técnica, matriz de impacto, critérios de aceitação em formato Given-When-Then, análise defensiva de segurança e plano shift-left de testes.
- `specs/frontend-web-app-delta.md`: Especificação arquitetural do Dashboard unificado, componente `EncyclopediaMenu`, padrões de UX Dark Fantasy para páginas "Em breve" e extensões de autenticação do cliente.
- `tasks.md`: Lista completa de tarefas atômicas para implementação, testes e validação.

### Critérios de Aceitação Definidos
1. **Visualização por Jogador**: Acessando `/characters` com papel `PLAYER`, visualiza "Meus Personagens" e categorias públicas de enciclopédia (Arquétipos, Magias, Talentos, Itens, Linhagens).
2. **Visualização por Mestre**: Acessando com papel `DUNGEON_MASTER`, visualiza adicionalmente os cards de Monstros e NPCs com a insígnia "Mestre".
3. **Páginas de Placeholder**: Usuários DM navegando para `/encyclopedia/monsters` ou `/encyclopedia/npcs` visualizam tela estilizada Dark Fantasy com status "Em breve" e botão de retorno.
4. **Proteção de Rota no Cliente**: Usuários apenas com `PLAYER` tentando acessar `/encyclopedia/monsters` ou `/encyclopedia/npcs` visualizam barreira de "Acesso restrito ao Mestre do Jogo".
5. **Redirecionamento Pós-Auth**: Conclusão de login ou cadastro redireciona para `/characters`.


---

## Fase 2: Extensão de Papéis no Auth (`useAuth`) e Verificação de Redirecionamentos

Nesta fase:
1. **Helper de Papéis em `useAuth`**:
   - Adicionadas as flags computadas `isDM` e `isPlayer`.
   - `isDM`: avalia reativamente se `user.roles` contém `"DUNGEON_MASTER"`.
   - `isPlayer`: avalia reativamente se `user.roles` contém `"PLAYER"`.
   - Fallback defensivo para `false` caso não autenticado, sem token ou caso `roles` seja vazio/nulo.
2. **Verificação de Redirecionamento**:
   - `login-form.tsx` e `register-form.tsx` foram verificados: ambos já utilizam `router.replace("/characters")` ao detectar sessão ativa e `router.push("/characters")` após submissão bem-sucedida.
3. **Testes Unitários (TDD)**:
   - Criado `__tests__/use-auth-roles.test.ts` cobrindo cenários TP-01, TP-02, TP-01b, TE-01 e TE-01b. Todos os 5 testes passaram com 100% de sucesso.
   - Suíte de 22 testes unitários do frontend executada com sucesso.


---

## Fase 3: Componente `EncyclopediaMenu` e Gating de Categorias por Papel

Nesta fase:
1. **Criação do Componente `EncyclopediaMenu` (`components/encyclopedia-menu.tsx`)**:
   - Desenvolvido seguindo os design tokens Dark Fantasy RPG (fundo `bg-zinc-900/60`, bordas sutis `border-zinc-800`, transição para hover `border-amber-500/40`, ícones Lucide).
   - Apresenta as 5 categorias públicas do compêndio: Arquétipos, Magias, Talentos, Itens e Linhagens.
   - Aplica controle de acesso condicional via `useAuth().isDM` para as categorias Monstros e NPCs.
   - Exibe a insígnia dourada "Mestre" com ícone `Crown` nos cards exclusivos de DM.
2. **Testes Unitários (TDD)**:
   - Criado `__tests__/encyclopedia-menu.test.tsx` cobrindo cenários TP-03, TP-04 e TE-02.
   - Validados a omissão de categorias restritas para jogadores comuns, a inclusão completa com badges para DMs e os destinos de rota de cada link.
   - 3 testes executados com 100% de aprovação.


---

## Fase 4: Páginas Placeholder de Monstros e NPCs ("Em breve") com Route Guard

Nesta fase:
1. **Criação das Páginas Placeholder**:
   - `/encyclopedia/monsters/page.tsx`: Bestiário & Monstros exclusivo para Mestres. Apresenta card Dark Fantasy estilizado com ícone `Skull`, badge "Em breve" e descrição sobre o catálogo SRD de criaturas, com botão de retorno ao Dashboard.
   - `/encyclopedia/npcs/page.tsx`: Galeria de NPCs para Mestres com ícone `UserCheck`, badge "Em breve" e descrição sobre gestão de NPCs para campanhas.
2. **Defensive Route Guarding no Cliente**:
   - Ambas as rotas verificam `isDM` via `useAuth()`.
   - Caso um usuário com papel apenas `PLAYER` tente acessar a URL diretamente, visualiza uma tela de advertência estilizada com `ShieldAlert`: *"Acesso Restrito ao Mestre do Jogo"* e botão para retornar a `/characters`.
3. **Testes Unitários (TDD)**:
   - Criado `__tests__/encyclopedia-placeholders.test.tsx` com 4 testes abrangendo acesso autorizado (DM) e bloqueio defensivo (Player) para ambas as páginas.
   - Suíte de 29 testes do frontend executada com sucesso total.
