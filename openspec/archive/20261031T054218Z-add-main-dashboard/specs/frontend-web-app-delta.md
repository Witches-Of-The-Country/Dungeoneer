# Delta Spec: `frontend-web-app`

> **Change ID:** `add-main-dashboard`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Unified Post-Login Dashboard (`/characters`)
- A rota `/characters` atua como hub principal pós-login, organizada em seções:
  - **Meus Personagens**: listagem de cards de personagens já existentes, botão de criar personagem, status e paginação/carregamento.
  - **Enciclopédia & Referências**: grid com cards de navegação para compêndios de regras e criação.

### 2. Client-Side Role Gating (`useAuth`)
- O hook `useAuth` fornece abstrações reativas para inspeção de papéis:
  - `isDM`: booleano indicando presença da role `DUNGEON_MASTER`.
  - `isPlayer`: booleano indicando presença da role `PLAYER`.
  - Tratamento resiliente para estados não autenticados ou listas vazias.

### 3. Encyclopedia Navigation Component (`EncyclopediaMenu`)
- Renderização em grade responsiva (1 coluna em telas menores, 2 a 4 colunas em telas maiores).
- Categorias padrão disponíveis a todos os usuários:
  - **Arquétipos (Classes)** (`/encyclopedia/classes`)
  - **Magias** (`/encyclopedia/spells`)
  - **Talentos** (`/encyclopedia/feats`)
  - **Itens** (`/encyclopedia/items`)
  - **Linhagens** (`/encyclopedia/lineages`)
- Categorias exclusivas para Mestres (`DUNGEON_MASTER`):
  - **Monstros** (`/encyclopedia/monsters`)
  - **NPCs** (`/encyclopedia/npcs`)
- Indicadores visuais: badge "Mestre" destacada com dourado e ícone de distinção.

### 4. Empty State UX Standards ("Em breve")
- Páginas de destino para funcionalidades em desenvolvimento (`/encyclopedia/monsters`, `/encyclopedia/npcs`):
  - Dark Fantasy styling (card escurecido, ícone temático, tipografia estilizada).
  - Mensagem explicativa amigável informando a previsão de disponibilidade.
  - Botão de navegação rápido para retornar ao Dashboard principal.
  - Guard de segurança na tela: se acessado diretamente por jogador sem role `DUNGEON_MASTER`, exibe barreira visual com "Acesso restrito ao Mestre do Jogo".

### 5. Navbar Integration
- Exibição de atalho ou menu dropdown para enciclopédia.
- Selo de identificação de "Mestre" adjacente ao nome/avatar quando o usuário possuir o papel `DUNGEON_MASTER`.
