# Handoff Document — Dungeoneer: `character-crud-backend`

> **Documento de Transição para Agente com Contexto Limpo**  
> **Data:** 2026-10-30  
> **Status do Projeto:** `auth-jwt-backend` e `reference-data-backend` concluídas e arquivadas. Próxima spec: `character-crud-backend`.

---

## 1. Visão Geral do Projeto e Arquitetura

O **Dungeoneer** é uma aplicação web para D&D 5e / 2024 (gestão de fichas de personagens, campanhas e sessões de jogo).  
A arquitetura é baseada em **Microsserviços** e segue rigorosamente os princípios de **Clean Architecture (Arquitetura Hexagonal)**.

### Estrutura de Camadas (Hexagonal) no Spring Boot:
```
domain/model/         → Entidades, Value Objects, Domain Interfaces (ZERO imports de framework: sem Spring, sem JPA, sem Lombok)
application/port/in/  → Interfaces de Use Cases (inbound ports)
application/port/out/ → Interfaces de Repositórios de saída (outbound ports)  
application/service/  → Implementações dos Use Cases
application/dto/      → Records / DTOs (com Jakarta Validation)
adapter/in/web/       → Controllers REST + Mappers Web + Error Handling (RFC 7807)
adapter/out/persistence/ → Entidades JPA + Repositórios Spring Data + Adaptadores Hexagonais
```

### Regras Globais Inegociáveis:
1. **Zero Framework no Domínio:** O pacote `domain` **NUNCA** deve conter anotações ou imports do Spring, JPA/Hibernate ou Lombok.
2. **Maven Global:** Sempre execute `mvn` (ex: `mvn -f player-character-service/backend/pom.xml test`), **NUNCA** use o wrapper (`./mvnw`).
3. **Cobertura JaCoCo:** Mínimo de **80% de cobertura de linha** nas camadas de domínio e aplicação.
4. **TDD Estrito (Red ➔ Green ➔ Refactor):** Todo código de produção deve nascer acompanhado de seus testes automatizados (JUnit 5 + Mockito para unitários, Testcontainers para persistência e MockMvc para endpoints).
5. **Idioma:** Código, nomes de variáveis, classes e documentação técnica em **inglês**. Interações com o usuário e o walkthrough cumulativo em **português (pt-BR)**.
6. **Pausas de Commit:** Execute uma fase por vez. Ao final de cada fase, forneça o comando de commit pronto (estilo Conventional Commits) e aguarde aprovação do usuário.

---

## 2. Status Atual da Esteira de Mudanças (SDD Pipeline)

| # | Spec / Change | Status | Localização | Notas |
|---|---|---|---|---|
| 1 | `auth-jwt-backend` | ✅ **Arquivada** | `openspec/archive/20261030T042203Z-auth-jwt-backend/` | JWT Auth, login, registro, cookies/localStorage no frontend |
| 2 | `reference-data-backend` | ✅ **Arquivada** | `openspec/archive/20261030T112901Z-reference-data-backend/` | Schema `player_character`, sementes V2/V3 (13 classes, 9 raças, 6 antecedentes para 2014 e 2024), endpoints `/api/reference/**` |
| 3 | **`character-crud-backend`** | 🟡 **AUDITADA & APROVADA (Próxima)** | `openspec/changes/character-crud-backend/` | CRUD completo com `X-User-Id`, rolagem de atributos `4d6kh3` e PV, cálculo de HP 1–20, regras 2014 vs 2024 |
| 4 | `gateway-routing-and-cors` | ⏳ Pendente | `openspec/changes/gateway-routing-and-cors/` | Roteamento do Gateway e injeção do header verificado `X-User-Id` |
| 5 | `character-frontend` | ⏳ Pendente | `openspec/changes/character-frontend/` | Wizard de criação de personagens no Next.js 16 com `@dnd-kit/core` e ficha |

---

## 3. Estado Atual do Ambiente & Git

### Estado do Repositório:
- **Repositório Root:** Branch `dev`
- **Submodule `player-character-service`:** Branch `feat/character-creation`

### Commits Prontos para Referência / Registro:
Se houver alterações pendentes no workspace da transição de arquivamento:
```powershell
# No submodule player-character-service:
cd player-character-service
git add .
git commit -m "feat(reference-data): implement reference data REST API, migrations, and error handling"
cd ..

# No repositório principal:
git add .
git commit -m "chore(spec): archive reference-data-backend and consolidate player-character-data-model v1.1.0"
```

### Docker & Banco de Dados:
- Todos os containers estão em execução (`docker-compose.yml`):
  - `db`: PostgreSQL 15 na porta `5432` (banco `dungeoneer`, usuário `postgres`, senha no `.env`).
  - `player-character-service`: porta `8080` (Spring Boot 4 / Java 21).
  - `auth-service`: porta `8082` (Spring Boot 4 / Java 21).
  - `dungeoneer-frontend`: porta `3000` (Next.js 16).
  - `api-gateway`: porta `9090` (Spring Cloud Gateway).
- **Nota sobre o host Windows:** Existe um serviço nativo `postgresql-x64-17` instalado no Windows. Caso precise rodar comandos locais diretamente contra o Docker via pgAdmin ou ferramentas do host, certifique-se de que o serviço do Windows está parado (`Stop-Service postgresql-x64-17`) ou use a porta mapeada.
- **Schema PostgreSQL:** O schema isolado **`player_character`** já existe no banco `dungeoneer` com as tabelas:
  - `player_character.archetypes` (26 registros: 13 do SRD 2014 e 13 do SRD 2024)
  - `player_character.lineages` (18 registros: 9 do SRD 2014 e 9 do SRD 2024)
  - `player_character.backgrounds` (12 registros: 6 do SRD 2014 e 6 do SRD 2024)
  - `player_character.characters`
  - `player_character.ability_scores`
- **IDs de Referência Semeados (UUIDs determinísticos fixos):**
  - Classes 2014: `10000000-0000-0000-0000-000000000001` até `0013`
  - Classes 2024: `20000000-0000-0000-0000-000000000001` até `0013`
  - Linhagens 2014: `11000000-0000-0000-0000-000000000001` até `0009`
  - Linhagens 2024: `21000000-0000-0000-0000-000000000001` até `0009`
  - Antecedentes 2014: `12000000-0000-0000-0000-000000000001` até `0006`
  - Antecedentes 2024: `22000000-0000-0000-0000-000000000001` até `0006`

---

## 4. A Especificação a Ser Executada: `character-crud-backend`

### Artefatos Centrais:
- **Proposta:** [`openspec/changes/character-crud-backend/proposal.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/proposal.md)
- **Tarefas:** [`openspec/changes/character-crud-backend/tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/tasks.md)
- **Delta do Domínio:** [`openspec/changes/character-crud-backend/specs/player-character-domain-delta.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/specs/player-character-domain-delta.md)
- **Delta do Banco:** [`openspec/changes/character-crud-backend/specs/player-character-data-model-delta.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/specs/player-character-data-model-delta.md)
- **Prompt Orquestrador das Fases:** [`prompt-character-crud-backend.md`](file:///D:/Projects/Dungeoneer/prompt-character-crud-backend.md)
- **Resultado da Auditoria Prévia:** **Aprovada sem reservas** pelo `sdd-spec-auditor`.

---

## 5. Roteiro de Execução das 6 Fases (Seguir Exatamente a Ordem)

O agente deve ler [`prompt-character-crud-backend.md`](file:///D:/Projects/Dungeoneer/prompt-character-crud-backend.md) e executar as fases sequencialmente:

### 🔹 Fase 1: Migração de Banco de Dados Flyway & Limpeza de Stubs
- **O que fazer:**
  1. Criar `V4__add_character_crud_and_roll_tables.sql` em `player-character-service/backend/src/main/resources/db/migration/`:
     - `ALTER TABLE player_character.characters`:
       - `ADD COLUMN notes TEXT NOT NULL DEFAULT ''`
       - `ADD COLUMN hp_calculation_method VARCHAR(15) NOT NULL DEFAULT 'MAX'`
       - `ADD COLUMN has_pending_level_choices BOOLEAN NOT NULL DEFAULT FALSE`
       - `ALTER COLUMN user_id SET NOT NULL`
     - `CREATE TABLE player_character.class_progressions`:
       - `id UUID PRIMARY KEY`, `character_id UUID NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE`, `archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id)`, `subclass_id UUID NULL`, `class_level INT NOT NULL CHECK (class_level >= 1 AND class_level <= 20)`, `UNIQUE (character_id, archetype_id)`.
     - `CREATE TABLE player_character.ability_rolls`:
       - `id UUID PRIMARY KEY`, `user_id UUID NOT NULL`, `rolled_scores INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
       - Índice parcial: `CREATE UNIQUE INDEX idx_unused_ability_roll ON player_character.ability_rolls(user_id) WHERE used = FALSE;`.
     - `CREATE TABLE player_character.hp_rolls`:
       - `id UUID PRIMARY KEY`, `user_id UUID NOT NULL`, `archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id)`, `target_level INT NOT NULL CHECK (target_level >= 2 AND target_level <= 20)`, `rolled_hp_per_level INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
       - Índice parcial: `CREATE UNIQUE INDEX idx_unused_hp_roll ON player_character.hp_rolls(user_id, archetype_id, target_level) WHERE used = FALSE;`.
  2. Limpar stubs dorminhocos/duplicados:
     - Deletar `CreatePlayerCharacterUseCase.java`, `CharacterRepositoryPort.java`, `CreatePlayerCharacterService.java`, `PlayerCharacterUseCaseConfig.java`.
- **Validação:** Rodar `mvn -f player-character-service/backend/pom.xml test` (78 testes passando).
- **Pausa de Commit:** Criar/atualizar `openspec/changes/character-crud-backend/walkthrough.md` e pausar para commit do usuário.

---

### 🔹 Fase 2: Camada de Domínio Pura & Testes Unitários TDD
- **O que fazer:**
  1. `ClassProgression`: entidade de domínio imutável com validação de nível 1–20.
  2. `AbilityScoreValidator`:
     - `STANDARD_ARRAY`: valida permutação exata de `[15, 14, 13, 12, 10, 8]`.
     - `POINT_BUY`: cada atributo entre 8 e 15, custo padrão de 27 pontos exatos.
     - `ROLLED`: validação de permutação multiset exata em relação ao roll persistido.
  3. `RulesEngine` (`Srd2014RulesEngine` e `Srd2024RulesEngine`):
     - `SRD_2014`: soma bônus fixos de linhagem (`Lineage.abilityBonuses`).
     - `SRD_2024`: valida se bônus de antecedentes (`+2/+1` ou `+1/+1/+1`) estão em `Background.allowedAbilityBonuses`, limitando em 20.
     - Cálculo de HP no servidor: Nível 1 = `hitDice.max + modCON` (mínimo 1). Níveis 2..N = `MAX`, `AVERAGE` (`d/2 + 1 + modCON`) ou `ROLLED` (`dadoRolado + modCON`).
     - Estatísticas derivadas: CA (10 + modDES), Iniciativa (modDES), Bônus de Proficiência por nível total, Velocidade (da Linhagem).
  4. Atualizar `PlayerCharacter`: adicionar `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, lista de `ClassProgression`, e método `updateSheet(...)` protegendo campos imutáveis.
  5. Exceções de domínio em `com.dungeoneer.playerCharacter.domain.exception`:
     - `CharacterNotFoundException`, `RollConflictException`, `InvalidRollTokenException`.
- **Validação:** Testes unitários puros com JUnit 5 cobrindo todos os cenários de domínio.
- **Pausa de Commit:** Atualizar walkthrough e pausar para commit.

---

### 🔹 Fase 3: Camada de Aplicação & Portas Hexagonais
- **O que fazer:**
  1. Portas de entrada (`application/port/in/`):
     - `CreateCharacterUseCase`, `GetCharacterUseCase`, `ListCharactersUseCase`, `UpdateCharacterUseCase`, `DeleteCharacterUseCase`.
     - `RollAbilityScoresUseCase`, `RollHitPointsUseCase`.
  2. Portas de saída (`application/port/out/`):
     - `PlayerCharacterRepository`: métodos com escopo de usuário (`findByIdAndUserId`, `findAllByUserId`, `deleteByIdAndUserId`).
     - `AbilityRollRepository`, `HpRollRepository`.
  3. DTOs em `application/dto/`:
     - `CharacterSheetDto`, `UpdateCharacterRequestDto`, `AbilityRollResponseDto`, `HpRollResponseDto`.
  4. Implementação dos serviços de aplicação em `application/service/` com `@Transactional`.
- **Validação:** Testes unitários com Mockito para todos os serviços de aplicação.
- **Pausa de Commit:** Atualizar walkthrough e pausar para commit.

---

### 🔹 Fase 4: Adaptadores de Persistência JPA & Testcontainers
- **O que fazer:**
  1. Entidades JPA mapeadas para `schema = "player_character"`:
     - `ClassProgressionJpaEntity`, `AbilityRollJpaEntity`, `HpRollJpaEntity`.
  2. Atualizar `PlayerCharacterJpaEntity`:
     - Adicionar `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, `@OneToMany` com `ClassProgressionJpaEntity`.
  3. Repositórios Spring Data JPA com queries escopadas por `userId`.
  4. Adaptadores de persistência implementando as portas de saída.
  5. Testes de integração com Testcontainers:
     - Idempotência de rolagem (chamar duas vezes retorna o mesmo registro não usado).
     - Consumo atômico da rolagem na criação do personagem (`used = true`).
     - Persistência e recuperação de fichas completas.
- **Validação:** `mvn -f player-character-service/backend/pom.xml test`.
- **Pausa de Commit:** Atualizar walkthrough e pausar para commit.

---

### 🔹 Fase 5: Adaptador Web REST & Testes MockMvc
- **O que fazer:**
  1. `PlayerCharacterController` em `/api/characters`:
     - `POST /api/characters`: cria personagem validando regras e consumindo roll se aplicável.
     - `POST /api/characters/ability-rolls`: gera ou retorna rolagem 4d6kh3 existente.
     - `POST /api/characters/hp-rolls`: gera ou retorna rolagem de PV existente.
     - `GET /api/characters`: lista personagens do usuário ordenados por `updated_at DESC`.
     - `GET /api/characters/{id}`: retorna ficha detalhada; `404` se for de outro usuário.
     - `PUT /api/characters/{id}`: atualiza somente nome, PV atual, PV temp e notas.
     - `DELETE /api/characters/{id}`: deleção física em cascata com retorno `204 No Content`.
     - **Obrigatório:** extrair `@RequestHeader("X-User-Id") UUID userId` em todas as rotas.
  2. Atualizar `GlobalExceptionHandler`:
     - Mapear `CharacterNotFoundException` ➔ `404 Not Found`.
     - Mapear `CharacterValidationException` ➔ `422 Unprocessable Entity`.
     - Mapear `RollConflictException` ➔ `409 Conflict`.
     - Mapear ausência de `X-User-Id` ➔ `401 Unauthorized` ou `400 Bad Request`.
  3. Testes MockMvc cobrindo TP-02, TP-03, TE-01, TE-02, TE-03 (incluindo mitigação de IDOR/BOLA e adulteração de atributos).
- **Validação:** Testes MockMvc passando com 100% de sucesso.
- **Pausa de Commit:** Atualizar walkthrough e pausar para commit.

---

### 🔹 Fase 6: Cobertura JaCoCo ≥ 80%, Consolidação de Specs e Arquivamento
- **O que fazer:**
  1. Rodar `mvn -f player-character-service/backend/pom.xml clean verify` e inspecionar o relatório JaCoCo (`target/site/jacoco/index.html`).
  2. Consolidar os deltas (`player-character-domain-delta.md` e `player-character-data-model-delta.md`) nas especificações canônicas correspondentes em `openspec/specs/`.
  3. Validar se todas as tarefas em `openspec/changes/character-crud-backend/tasks.md` estão marcadas como `[x]`.
  4. Executar o arquivamento via `sdd-spec-archive`, movendo para `openspec/archive/<timestamp>-character-crud-backend/`.
  5. Fornecer os comandos de commit finais.

---

## 6. Primeiro Comando Recomendado para o Novo Agente

O novo agente deve iniciar imediatamente pela **Fase 1**:
1. Ler [`prompt-character-crud-backend.md`](file:///D:/Projects/Dungeoneer/prompt-character-crud-backend.md) linhas 23 a 62.
2. Criar a migração `V4__add_character_crud_and_roll_tables.sql` em `player-character-service/backend/src/main/resources/db/migration/`.
3. Deletar os stubs dorminhocos em `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/`.
4. Rodar `mvn -f player-character-service/backend/pom.xml test` para garantir que o projeto compila e passa nos 78 testes existentes.
5. Iniciar o arquivo `openspec/changes/character-crud-backend/walkthrough.md` e apresentar a pausa para commit da Fase 1 ao usuário.
