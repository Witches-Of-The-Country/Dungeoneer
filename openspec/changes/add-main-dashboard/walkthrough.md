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
