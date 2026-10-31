# Proposal: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`
> **Status:** in-progress
> **Author:** jaca
> **Created:** 2026-10-31
> **Last Updated:** 2026-10-31

---

## Summary

Telas e endpoints de consulta paginada por cursor para compêndios de regras de D&D (Arquétipos, Magias, Talentos, Itens e Linhagens) com carregamento preguiçoso (*infinite scroll* via TanStack Query), busca por texto em linguagem natural (PostgreSQL Full-Text Search com índices GIN e dicionário em português) e filtros contextuais por regras (`SRD_2014`, `SRD_2024`), círculo/escola de magia e raridade/tipo de item. Inclui a ingestão (*seed*) de conteúdo oficial do SRD 5.1 e 5.2 traduzido para pt-BR em migrações Flyway versionadas e idempotentes.

## Motivation

Atualmente, jogadores e mestres possuem apenas visualizações estáticas ou incompletas de regras no assistente de criação de fichas. O compêndio de referência precisa ser consultado de forma rápida, eficiente e responsiva na interface web sem travar o navegador ou sobrecarregar a memória do servidor com paginações `OFFSET` ineficientes. A adição do banco traduzido do SRD para Magias, Talentos e Itens viabiliza um ambiente completo de pesquisa e referência oficial sem depender de APIs externas.

## Scope

### In Scope
- Endpoints REST paginados por cursor com filtros e parâmetro de busca (`q`) na API de dados de referência (`ReferenceDataController`).
- Criação de tabelas relacionais `player_character.spells`, `player_character.feats` e `player_character.items` com colunas `slug`, `name_en`, `name` (pt-BR), `description` (pt-BR) e `UNIQUE (ruleset, slug)`.
- Índices PostgreSQL Full-Text Search GIN usando `to_tsvector('portuguese', ...)` e índices B-Tree de suporte ao cursor composto `(name, id)` e filtros.
- Modelo de domínio puro `Item.java` em `domain/model/` (Clean Architecture, zero anotações de framework) e repositórios JPA correspondentes.
- **Seed de conteúdo SRD traduzido (pt-BR)** para Magias, Talentos e Itens (equipamentos, armas, armaduras e itens mágicos), cobrindo SRD 5.1 (`SRD_2014`) e SRD 5.2 (`SRD_2024`), injetado via migrations Flyway versionadas (V7 cria as tabelas; V8 spells, V9 feats, V10 items), idempotentes por `(ruleset, slug)`.
- Arquivos de dados de origem versionados em JSON (`resources/db/seed/`) mantendo `name_en` e terminologia oficial de D&D em pt-BR.
- Frontend: cliente de API e hooks customizados com TanStack Query `useInfiniteQuery` (`lib/api/encyclopedia.ts`).
- Frontend: telas de consulta Next.js 16 para Classes/Arquétipos, Magias, Talentos, Itens e Linhagens com campo de busca com *debounce*, seletores de filtro, scroll infinito (*IntersectionObserver*), skeletons de carregamento e estado vazio no estilo Dark Fantasy.
- Rodapé de atribuição da licença Creative Commons Attribution 4.0 International (CC-BY 4.0) nas telas da enciclopédia.
- Suíte completa de testes automatizados (unitários, MockMvc, Testcontainers PostgreSQL e Vitest).

### Out of Scope
- Criação de conteúdo homebrew (escopo reservado para a change subsequente `add-homebrew-creation`).
- Ferramentas ativas de Monstros e NPCs (permanecem em modo placeholder até módulo de campanhas).
- Seed de Arquétipos e Linhagens (já populados no banco nas migrações V1–V6).
- Edição em tempo de execução ou vínculo dinâmico com mecânicas de fichas nesta entrega (o compêndio atua como texto estático de consulta).

## Proposed Solution

### Approach
1. **Modelagem Relacional e Busca Textual (PostgreSQL)**:
   - Migração Flyway `V7__create_spells_feats_items_and_fts_indexes.sql`:
     - Tabelas `spells`, `feats` e `items` sob o schema `player_character`.
     - Identificadores únicos com chave primária UUID e chave natural composta `UNIQUE (ruleset, slug)`.
     - Colunas `name_en` (nome oficial original) e `name`/`description` (tradução pt-BR).
     - Índices GIN com `to_tsvector('portuguese', coalesce(name, '') || ' ' || coalesce(name_en, '') || ' ' || coalesce(description, ''))` garantindo busca rápida tanto por termos em português quanto em inglês.
     - Índices B-Tree compostos para cursor de paginação estável `(name, id)` e filtros frequentes.
2. **Migrations de Seed do SRD Traduzido**:
   - `V8__seed_srd_spells.sql`: Magias do SRD 5.1 e SRD 5.2.
   - `V9__seed_srd_feats.sql`: Talentos do SRD 5.1 e SRD 5.2.
   - `V10__seed_srd_items.sql`: Equipamentos, armas, armaduras e itens mágicos do SRD 5.1 e 5.2.
   - Idempotência absoluta garantida por `ON CONFLICT (ruleset, slug) DO NOTHING`.
3. **Clean Architecture no Backend**:
   - Entidade de domínio pura `Item.java` sem dependências do Spring ou Jakarta.
   - Porta de saída `ReferenceDataRepository` expondo consultas paginadas por cursor.
   - DTO genérico `CursorPageResponse<T>` com estratégia de leitura `limit + 1` para derivação precisa de `hasNext` e `nextCursor`.
4. **Interface Reativa com TanStack Query**:
   - Hooks `useInfiniteSpells`, `useInfiniteFeats`, `useInfiniteItems`, etc., gerenciando cache, busca com debounce e paginação contínua.
   - Interface Dark Fantasy responsiva com feedback visual imediato de filtros e carregamento.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V7__create_spells_feats_items_and_fts_indexes.sql` | Create | Tabelas spells/feats/items + índices GIN tsvector |
| `player-character-service/backend/src/main/resources/db/migration/V8__seed_srd_spells.sql` | Create | Seed SRD 2014/2024 de magias traduzidas |
| `player-character-service/backend/src/main/resources/db/migration/V9__seed_srd_feats.sql` | Create | Seed SRD 2014/2024 de talentos traduzidos |
| `player-character-service/backend/src/main/resources/db/migration/V10__seed_srd_items.sql` | Create | Seed SRD 2014/2024 de itens e equipamentos traduzidos |
| `player-character-service/backend/src/main/resources/db/seed/` | Create | Arquivos JSON de dados de referência do seed |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Item.java` | Create | Entidade pura de domínio para itens |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpellJpaEntity.java` | Create | JPA Entity para tabela spells |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntity.java` | Create | JPA Entity para tabela feats |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ItemJpaEntity.java` | Create | JPA Entity para tabela items |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataSpellRepository.java` | Create | Repositório Spring Data de spells com FTS e cursor |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataFeatRepository.java` | Create | Repositório Spring Data de feats com FTS e cursor |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataItemRepository.java` | Create | Repositório Spring Data de items com FTS e cursor |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataArchetypeRepository.java` | Modify | Consultas paginadas por cursor e FTS |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataLineageRepository.java` | Modify | Consultas paginadas por cursor e FTS |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java` | Modify | Inclusão de métodos de consulta paginada por cursor |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java` | Modify | Implementação das consultas no adapter |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/GetReferenceDataUseCase.java` | Modify | Métodos de caso de uso com cursor e filtros |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/ReferenceDataService.java` | Modify | Orquestração de cursor e montagem de CursorPageResponse |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CursorPageResponse.java` | Create | Record genérico de resposta paginada |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/SpellDto.java` | Create | DTO de magias |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/FeatDto.java` | Create | DTO de talentos |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/ItemDto.java` | Create | DTO de itens |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java` | Modify | Endpoints paginados `/api/reference/*` |
| `player-character-service/backend/src/test/java/.../SeedMigrationTest.java` | Create | Testcontainers: validação de idempotência, contagens e FTS no seed |
| `dungeoneer-frontend/lib/api/encyclopedia.ts` | Create | Tipos TypeScript e hooks TanStack Query infinite |
| `dungeoneer-frontend/app/encyclopedia/layout.tsx` | Create | Layout compartilhado com tabs e footer CC-BY 4.0 |
| `dungeoneer-frontend/app/encyclopedia/classes/page.tsx` | Create | Página de consulta de arquétipos |
| `dungeoneer-frontend/app/encyclopedia/spells/page.tsx` | Create | Página de consulta de magias |
| `dungeoneer-frontend/app/encyclopedia/feats/page.tsx` | Create | Página de consulta de talentos |
| `dungeoneer-frontend/app/encyclopedia/items/page.tsx` | Create | Página de consulta de itens |
| `dungeoneer-frontend/app/encyclopedia/lineages/page.tsx` | Create | Página de consulta de linhagens |

## Acceptance Criteria

- [ ] **Given** a base de dados migrada, **When** as migrações V7, V8, V9 e V10 forem executadas, **Then** as tabelas `spells`, `feats` e `items` são criadas e preenchidas com os dados do SRD 2014 e 2024, mantendo unicidade de `(ruleset, slug)`.
- [ ] **Given** uma requisição para `GET /api/reference/spells?limit=10`, **When** executada sem cursor prévio, **Then** retorna os primeiros 10 itens em ordem alfabética estável, `hasNext: true` e o `nextCursor` preenchido.
- [ ] **Given** uma requisição contendo o `cursor` recebido anteriormente, **When** submetida para o endpoint, **Then** retorna a página consecutiva de itens sem duplicatas nem omissões.
- [ ] **Given** uma busca com parâmetro textual `q=míssil` ou `q=magic missile`, **When** processada pelo backend, **Then** localiza a magia correspondente utilizando o índice FTS GIN.
- [ ] **Given** a interface de consulta de magias no frontend, **When** o usuário digitar no campo de busca ou selecionar um filtro de círculo/escola, **Then** a lista atualiza reativamente com debounce de 300ms e carrega páginas subsequentes conforme o scroll atinge o fim da página.
- [ ] **Given** as páginas de enciclopédia, **When** renderizadas no navegador, **Then** exibem no rodapé a menção de conformidade com a licença CC-BY 4.0 do System Reference Document.

## Defensive Security & Validation Considerations

### Input Validation
- Validação estrita de limites de paginação via Jakarta Validation no controller: `@Min(1) @Max(100) int limit`, prevenindo ataques de negação de serviço por alocação excessiva de memória.
- Sanitização de parâmetros de busca Full-Text: uso de queries parametrizadas com `plainto_tsquery('portuguese', :q)` ou `websearch_to_tsquery`, impedindo SQL Injection através de operadores especiais de busca.

### Secret Isolation
- Nenhuma chave secreta ou credencial é exposta. Conexões de banco permanecem isoladas via injeção de variáveis de ambiente.

### Authorization & Authentication Impact
- Os endpoints de leitura de dados de referência são públicos para todos os usuários autenticados (`PLAYER` e `DUNGEON_MASTER`). O gateway valida o JWT e propaga o cabeçalho seguro downstream.

### Other Security Risks
- Prevenção de travamento por re-execução de migração: uso estrito de cláusulas idempotentes `ON CONFLICT DO NOTHING`.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Seed migrations (V7–V10) em banco limpo | Integration (Testcontainers) | Cria tabelas, índices e insere registros com contagem esperada |
| TP-02 | Consulta paginada inicial com limit=10 | Integration (MockMvc) | Retorna status 200, 10 itens e `hasNext: true` com cursor |
| TP-03 | Consulta paginada com cursor consecutivo | Integration (MockMvc) | Retorna itens seguintes mantendo integridade sequencial |
| TP-04 | Busca Full-Text por termo em português e inglês | Integration (Testcontainers) | Retorna registros relevantes por correspondência no nome ou descrição |
| TP-05 | Filtro de magias por regras (`SRD_2024`) e nível | Integration (MockMvc) | Retorna apenas magias que atendem a ambos os filtros |
| TP-06 | Renderização de lista infinita e debounce no frontend | Unit (Vitest) | Dispara queries apropriadas e renderiza cards temáticos |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Re-execução das migrações de seed (idempotência) | Integration (Testcontainers) | Nenhuma exceção de chave duplicada ou erro de integridade |
| TE-02 | Requisição com limit excessivo (> 100) | Integration (MockMvc) | Retorna HTTP 400 Bad Request com RFC 7807 ProblemDetail |
| TE-03 | Busca com caracteres especiais (`!`, `&`, `'`) | Integration (Testcontainers) | Não lança erro de sintaxe SQL/tsquery e busca de forma segura |
| TE-04 | Busca sem resultados | Unit / Integration | Retorna lista vazia, `hasNext: false` e UI exibe Empty State amigável |

### Test Commands
- **Backend tests:** `mvn -f player-character-service/backend/pom.xml clean test`
- **Backend verify & coverage:** `mvn -f player-character-service/backend/pom.xml clean verify`
- **Frontend tests:** `npm --prefix dungeoneer-frontend run test:unit`
- **Frontend build:** `npm --prefix dungeoneer-frontend run build`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Volume elevado de dados de migração de seed tornando o build lento | Média | Baixa | Divisão dos arquivos SQL em lotes estruturados e índices otimizados para rápida inserção |
| Ordem instável em cursores baseados em UUID puro | Alta | Alta | Cursor de paginação composto ordenado por `(name, id)` alfabético estável |
| Divergência de termos em inglês vs português | Baixa | Média | Inclusão de `name_en` tanto no schema quanto no `tsvector` de busca FTS |

## Specification Changes & Delta Stubs

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Adição das tabelas `spells`, `feats`, `items`, colunas `slug`, `name_en`, categoria de itens, índices GIN tsvector e constraints de unicidade.

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Adição da entidade pura `Item`, contratos de repositório paginado e contratos de use case para enciclopédia.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Rotas de consulta da enciclopédia (`/encyclopedia/*`), componentes de busca com debounce, infinite scroll e atribuição CC-BY 4.0.

## References
- `openspec/specs/player-character-data-model.md`
- `openspec/specs/player-character-domain.md`
- `openspec/specs/frontend-web-app.md`
- `prompt-02-add-encyclopedia-browse.md`
