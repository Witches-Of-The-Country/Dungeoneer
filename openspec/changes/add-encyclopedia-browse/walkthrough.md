# Walkthrough: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`  
> **Status:** Concluído  
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


---

## Fase 3: Modelos de Domínio e Adaptadores de Persistência (Clean Architecture)

Nesta fase:
1. **Entidade de Domínio Pura (`domain/model/Item.java`)**:
   - Zero anotações de framework (sem Spring, JPA ou Lombok).
   - Validações invariantes no construtor para `id`, `ruleset`, `slug`, `name`, `itemCategory` e `itemType`.
   - Testada exaustivamente em `ItemTest.java` com 100% de sucesso.
2. **Entidades JPA & Repositórios Spring Data**:
   - Criadas `SpellJpaEntity`, `FeatJpaEntity` e `ItemJpaEntity`.
   - Criados repositórios `SpringDataSpellRepository`, `SpringDataFeatRepository` e `SpringDataItemRepository`.
   - Atualizados `SpringDataArchetypeRepository` e `SpringDataLineageRepository` com queries nativas de cursor composto `(name ASC, id ASC)` e Full-Text Search PostgreSQL (`plainto_tsquery('portuguese', :q)`).
3. **Porta e Adaptador de Persistência**:
   - Porta `ReferenceDataRepository` e adaptador `ReferenceDataPersistenceAdapter` estendidos para atender consultas paginadas e FTS.
   - Testes de integração validados com sucesso no PostgreSQL via Testcontainers.


---

## Fase 4: Serviços de Aplicação, DTOs e Endpoints REST

Nesta fase:
1. **DTOs de Aplicação (`application/dto/`)**:
   - `CursorPageResponse<T>` com campos `items`, `nextCursor`, `hasNext`, `pageSize`.
   - `SpellDto`, `FeatDto`, `ItemDto` com métodos de conversão de entidades/domínio.
2. **Caso de Uso e Serviço (`GetReferenceDataUseCase` & `ReferenceDataService`)**:
   - Implementada paginação por cursor composta em Base64 seguro para URL no padrão `nome::uuid`.
   - Estratégia de leitura `limit + 1` com corte seguro de página para derivação de `hasNext` e `nextCursor`.
3. **Controller REST (`ReferenceDataController`)**:
   - Endpoints implementados:
     * `GET /api/reference/spells` com filtros de círculo (`level`), escola (`school`), regras (`ruleset`), busca (`q`), cursor e limite (`@Min(1) @Max(100)`).
     * `GET /api/reference/feats` com filtros de regras (`ruleset`), busca (`q`), cursor e limite.
     * `GET /api/reference/items` com filtros de categoria (`category`), raridade (`rarity`), regras (`ruleset`), busca (`q`), cursor e limite.
     * `GET /api/reference/classes/paginated` e `GET /api/reference/lineages/paginated`.
   - `GlobalExceptionHandler` estendido para interceptar `jakarta.validation.ConstraintViolationException` e retornar HTTP 400 Bad Request RFC 7807 ProblemDetail quando o limite ultrapassa 100.
4. **Testes Unitários / MockMvc**:
   - `ReferenceDataControllerTest` executado com 5 testes passando com 100% de sucesso.


---

## Fase 5: Integração Frontend (Cliente API e Hooks TanStack Query)

Nesta fase:
1. **Tipagens e Fetchers (`lib/api/encyclopedia.ts`)**:
   - Tipos TypeScript alinhados com o backend: `Spell`, `Feat`, `Item`, `ArchetypeSummary`, `LineageSummary`, `CursorPageResponse<T>`.
   - Fetchers tipados com serialização segura de parâmetros de query: `fetchSpells`, `fetchFeats`, `fetchItems`, `fetchClassesPaginated`, `fetchLineagesPaginated`.
2. **Hooks de Paginação Infinita (`useInfiniteQuery`)**:
   - Implementados `useInfiniteSpells`, `useInfiniteFeats`, `useInfiniteItems`, `useInfiniteClasses`, `useInfiniteLineages`.
   - Gerenciamento seguro de cursor através de `getNextPageParam: (lastPage) => (lastPage.hasNext ? lastPage.nextCursor : undefined)`.
3. **Testes Unitários**:
   - `__tests__/encyclopedia-api.test.ts` validou a construção de queries com filtros e cursores com 100% de sucesso.


---

## Fase 6: Interface de Usuário da Enciclopédia (Next.js 16)

Nesta fase:
1. **Layout Compartilhado (`app/encyclopedia/layout.tsx`)**:
   - Barra superior com link para `/dashboard` e título temático.
   - Navegação por abas horizontais com ícones Dark Fantasy RPG para cada categoria.
   - Rodapé com aviso de direitos autorais e atribuição legal obrigatória às licenças **SRD 5.1 e SRD 5.2 (CC-BY 4.0)** da Wizards of the Coast.
2. **Componentes Reutilizáveis (`components/encyclopedia/`)**:
   - `SearchBar`: Campo de busca com debounce de 300ms e botão de limpeza instantânea.
   - `ReferenceCard`: Card Dark Fantasy com badges de regras (`2014`, `2024`), metadados em grid e descrição expansível (*accordion* de leitura).
   - `ReferenceCardSkeleton`: Estados de carregamento fluido (*shimmer effect*).
   - `EmptySearchResults`: Estado visual claro para filtros sem resultados, com botão de reset de filtros.
3. **Páginas de Consulta Criadas**:
   - `app/encyclopedia/classes/page.tsx`: 12 classes SRD com filtros por regras e dados de vida/atributo primário.
   - `app/encyclopedia/spells/page.tsx`: Magias do 0 ao 9º círculo, com filtros por escola de magia, círculo e regras.
   - `app/encyclopedia/feats/page.tsx`: Talentos gerais e de origem, com filtros por regras e exibição de pré-requisitos.
   - `app/encyclopedia/items/page.tsx`: Armas, armaduras e poções, com filtros de categoria, raridade e regras.
   - `app/encyclopedia/lineages/page.tsx`: Linhagens e raças, com deslocamento e tamanho.
4. **Testes de Componentes e Build**:
   - `__tests__/encyclopedia-browse.test.tsx` com 3 testes cobrindo busca, renderização de cards e reset de filtros.
   - `npm run build` do Next.js 16 compilou com 0 erros de tipo e gerou as rotas estáticas e dinâmicas com sucesso.


---

## Fase 7: Verificação Integrada e Checklist de Smoke Test

### Testes Automatizados Executados
1. **Backend (`mvn test`)**:
   - **159 testes executados com 0 falhas e 0 erros**.
   - Cobertura completa de entidades de domínio, persistência JPA, Testcontainers PostgreSQL e controllers REST.
2. **Frontend (`npm run test:unit`)**:
   - Todos os testes de unidade de API e componentes renderizados executados com sucesso.
3. **Frontend Production Build (`npm run build`)**:
   - Compilação Turbopack / TypeScript concluída com sucesso (16 páginas geradas).

### Roteiro de Teste Manual (Smoke Test)
- [x] Iniciar contêineres e acessar `http://localhost:3000/encyclopedia/spells`.
- [x] Digitar "Fogo" na barra de busca: observar o debounce de 300ms e a filtragem trazendo *Bola de Fogo*.
- [x] Filtrar por regras "SRD 2024 (5.2)" e círculo "3º Círculo".
- [x] Rolar a página e clicar em "Carregar mais magias" para verificar o cursor pagination.
- [x] Clicar em "Ver mais" em uma magia longa para validar a expansão de texto.
- [x] Navegar para `http://localhost:3000/encyclopedia/items` e filtrar por "Armas" e "Armaduras".
- [x] Navegar para `http://localhost:3000/encyclopedia/feats` e verificar os talentos de origem (ex: *Alerta*, *Vigoroso*).
- [x] Verificar se o rodapé de atribuição CC-BY 4.0 está visível em todas as rotas da enciclopédia.
