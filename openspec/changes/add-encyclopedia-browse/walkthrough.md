# Walkthrough: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`  
> **Status:** Em andamento  
> **Data:** 31/10/2026  

---

## Fase 1: Finalização da Especificação e Gates SDD

Nesta fase inicial:
- Verificado pré-requisito: a change `add-main-dashboard` foi concluída e arquivada com sucesso em `openspec/archive/20261031T054218Z-add-main-dashboard/`, e a especificação canônica `openspec/specs/frontend-web-app.md` foi consolidada na versão `1.2.0`.
- Documentos de especificação completados em `openspec/changes/add-encyclopedia-browse/`:
  - `proposal.md`: Detalhando arquitetura de cursor pagination estável `(name, id)`, índices PostgreSQL GIN de texto completo para pt-BR e inglês, estratégias defensivas de limites de paginação (`@Max(100)`), plano de testes shift-left e o escopo de ingestão (*seed*) de conteúdo traduzido do SRD 5.1 e 5.2 (Magias, Talentos e Itens).
  - `specs/player-character-data-model-delta.md`: Definição das tabelas `spells`, `feats`, `items`, colunas `slug`, `name_en`, constraints `UNIQUE (ruleset, slug)` e índices FTS.
  - `specs/player-character-domain-delta.md`: Modelo de domínio `Item.java` e novos contratos de use case e repositório.
  - `specs/frontend-web-app-delta.md`: Rotas da enciclopédia, hooks TanStack Query e rodapé CC-BY 4.0.
  - `tasks.md`: Lista completa e atômica de tarefas de implementação, testes e validação.


---

## Fase 2: Migração de Banco (Flyway V7) e Índices Full-Text Search (GIN)

Nesta fase:
1. **Criação do Script Flyway `V7__create_spells_feats_items_and_fts_indexes.sql`**:
   - Criação das tabelas `player_character.spells`, `player_character.feats` e `player_character.items`.
   - Adicionadas colunas `slug VARCHAR(150)`, `name_en VARCHAR(150)`, `name VARCHAR(150)`, `description TEXT` e chave natural de unicidade `UNIQUE (ruleset, slug)`.
   - Na tabela `items`, adicionada a coluna discriminadora `item_category VARCHAR(30)` (`EQUIPMENT`, `WEAPON`, `ARMOR`, `MAGIC`) e `rarity VARCHAR(30)`.
   - Índices B-Tree compostos criados para ordenação alfabética e paginação por cursor: `(ruleset, name ASC, id ASC)`.
   - Índices GIN de texto completo (FTS) criados com dicionário em português: `to_tsvector('portuguese', coalesce(name, '') || ' ' || coalesce(name_en, '') || ' ' || coalesce(description, ''))`.
2. **Validação com Testcontainers PostgreSQL**:
   - O teste de integração `ReferenceDataPersistenceAdapterTest` rodou a migração V7 com sucesso total sobre o contêiner PostgreSQL do Docker.
