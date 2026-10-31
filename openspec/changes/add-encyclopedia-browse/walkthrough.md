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


---

## Fase 2.1: Seed de Magias SRD Traduzidas (Flyway V8)

Nesta fase:
1. **Dados de Origem em JSON**:
   - `player-character-service/backend/src/main/resources/db/seed/spells_srd_2014.json`
   - `player-character-service/backend/src/main/resources/db/seed/spells_srd_2024.json`
   - Campos armazenados: `ruleset`, `slug`, `name_en`, `name`, `level`, `school`, `casting_time`, `range_area`, `components`, `duration`, `description`.
2. **Migração Flyway V8**:
   - `V8__seed_srd_spells.sql` implementado com inserções idempotentes usando `ON CONFLICT (ruleset, slug) DO NOTHING` e dollar-quoting `$$...$$` nos textos das magias.
3. **Amostra de Tradução para Revisão**:
   - *Guidance* -> **Orientação** (Truque, Adivinhação)
   - *Light* -> **Luz** (Truque, Evocação)
   - *Mage Hand* -> **Mãos Mágicas** (Truque, Conjuração)
   - *Cure Wounds* -> **Curar Ferimentos** (1º Círculo, Evocação / Abjuração)
   - *Magic Missile* -> **Mísseis Mágicos** (1º Círculo, Evocação)
   - *Shield* -> **Escudo Arcano** (1º Círculo, Abjuração)
   - *Healing Word* -> **Palavra Curativa** (1º Círculo, Abjuração)
   - *Fireball* -> **Bola de Fogo** (3º Círculo, Evocação)
4. **Verificação Automatizada**:
   - Migração V8 executada e validada com sucesso via Testcontainers no PostgreSQL.


---

## Fase 2.2: Seed de Talentos SRD Traduzidos (Flyway V9)

Nesta fase:
1. **Dados de Origem em JSON**:
   - `player-character-service/backend/src/main/resources/db/seed/feats_srd_2014.json`
   - `player-character-service/backend/src/main/resources/db/seed/feats_srd_2024.json`
   - Campos armazenados: `ruleset`, `slug`, `name_en`, `name`, `prerequisites`, `description`.
2. **Migração Flyway V9**:
   - `V9__seed_srd_feats.sql` implementado com inserções idempotentes via `ON CONFLICT (ruleset, slug) DO NOTHING`.
3. **Amostra de Tradução para Revisão**:
   - *Grappler* (2014) -> **Imobilizador** (Pré-requisito: Força 13+)
   - *Alert* (2024) -> **Alerta** (Talento de Origem)
   - *Magic Initiate* (2024) -> **Iniciado em Magia** (Talento de Origem)
   - *Tough* (2024) -> **Vigoroso** (Talento de Origem)
   - *Savage Attacker* (2024) -> **Atacante Selvagem** (Talento de Origem)
   - *War Caster* (2024) -> **Conjurador de Batalha** (Pré-requisito: Nível 4+)
4. **Verificação Automatizada**:
   - Migração V9 testada e executada com sucesso via Testcontainers no PostgreSQL.


---

## Fase 2.3: Seed de Itens e Equipamentos SRD Traduzidos (Flyway V10)

Nesta fase:
1. **Dados de Origem em JSON**:
   - `player-character-service/backend/src/main/resources/db/seed/items_srd_2014.json`
   - `player-character-service/backend/src/main/resources/db/seed/items_srd_2024.json`
   - Campos armazenados: `ruleset`, `slug`, `name_en`, `name`, `item_category`, `item_type`, `rarity`, `attunement`, `cost`, `weight`, `description`.
2. **Migração Flyway V10**:
   - `V10__seed_srd_items.sql` implementado com inserções idempotentes via `ON CONFLICT (ruleset, slug) DO NOTHING`.
3. **Amostra de Tradução para Revisão**:
   - *Longsword* (2014) -> **Espada Longa** (Arma Marcial Corpo a Corpo, dano 1d8/1d10 versátil)
   - *Chain Mail* (2014) -> **Cota de Malha** (Armadura Pesada, CA 16)
   - *Potion of Healing* (2014) -> **Poção de Cura** (Poção Mágica Comum, cura 2d4+2)
   - *Boots of Elvenkind* (2014) -> **Botas Élficas** (Item Maravilhoso Incomum, passos silenciosos)
   - *Longsword* (2024) -> **Espada Longa** (Arma Marcial Corpo a Corpo com maestria Sap/Abalar)
   - *Plate Armor* (2024) -> **Armadura de Placas** (Armadura Pesada, CA 18)
   - *Potion of Healing* (2024) -> **Poção de Cura** (Regras 2024 com consumo como Ação Bônus)
   - *Cloak of Protection* (2024) -> **Manto de Proteção** (Item Maravilhoso Incomum, +1 CA e Salvaguardas)
4. **Verificação Automatizada**:
   - Migração V10 testada e executada com sucesso via Testcontainers no PostgreSQL.
