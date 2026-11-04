# Walkthrough — Implementação do Character CRUD Backend (`character-crud-backend`)

Este documento consolida o progresso cumulativo das fases de implementação da especificação `character-crud-backend` no serviço `player-character-service`.

---

## Fase 1: Database Migration & Remoção de Stubs Inativos

### 1. Modificações Realizadas
- **Migração Flyway criada**:
  - [`V4__add_character_crud_and_roll_tables.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V4__add_character_crud_and_roll_tables.sql):
    - Adicionadas colunas à tabela `player_character.characters`: `notes`, `hp_calculation_method`, `has_pending_level_choices`, restrição `user_id NOT NULL` e índice `idx_characters_user_id`.
    - Criada a tabela `player_character.class_progressions` com FK em cascata para `characters(id)` e `archetypes(id)`, restrição de unicidade `(character_id, archetype_id)` e índice.
    - Criada a tabela `player_character.ability_rolls` com array de inteiros `rolled_scores`, breakdown em JSONB `dice_breakdown` e índice parcial único `idx_unused_ability_roll` em `(user_id) WHERE used = FALSE`.
    - Criada a tabela `player_character.hp_rolls` com `archetype_id`, `target_level` (2–20), `rolled_hp_per_level INT[]` e índice parcial único `idx_unused_hp_roll` em `(user_id, archetype_id, target_level) WHERE used = FALSE`.
- **Limpeza de Stubs Inativos**:
  - Removidos stubs órfãos: `CreatePlayerCharacterUseCase`, `CharacterRepositoryPort`, `CreatePlayerCharacterService`, `PlayerCharacterUseCaseConfig`.

### 2. Resultados de Verificação
- `mvn -f player-character-service/backend/pom.xml test` -> `BUILD SUCCESS` (78 testes).

---

## Fase 2: Camada de Domínio Pura & Testes Unitários (TDD)

### 1. Modificações Realizadas
- **Modelos de Domínio e Value Objects**:
  - [`HpCalculationMethod.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/HpCalculationMethod.java): Enum com `MAX`, `AVERAGE`, `ROLLED`.
  - [`ClassProgression.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ClassProgression.java): Entidade pura para rastreamento de classe e nível (1–20), subclasse e validações invariantes.
  - [`AbilityScoreValidator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/AbilityScoreValidator.java): Validador para `STANDARD_ARRAY`, `POINT_BUY` (27 pontos) e `ROLLED` (permutação de multiconjunto dos valores rolados).
  - [`RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RulesEngine.java), [`Srd2014RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2014RulesEngine.java) e [`Srd2024RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2024RulesEngine.java): Implementação da mecânica de Dual Ruleset (ASI de linhagem em 2014; ASI de antecedente +2/+1 ou +1/+1/+1 em 2024 com teto de 20; cálculo de HP inicial para níveis 1–20 por MAX, AVERAGE ou ROLLED; CA, iniciativa, proficiência e deslocamento).
  - [`PlayerCharacter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java): Atualizado com `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, `classProgressions` e o método `updateSheet(...)` protegendo atributos imutáveis.
- **Exceções de Domínio**:
  - [`CharacterNotFoundException.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/exception/CharacterNotFoundException.java)
  - [`RollConflictException.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/exception/RollConflictException.java)
  - [`InvalidRollTokenException.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/exception/InvalidRollTokenException.java)
- **Suíte de Testes Unitários de Domínio**:
  - [`AbilityScoreValidatorTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/AbilityScoreValidatorTest.java) (TP-01, TE-01)
  - [`RulesEngineTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/RulesEngineTest.java) (TP-01, TE-03)
  - [`PlayerCharacterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacterTest.java)

### 2. Resultados de Verificação
- `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.domain.**.*Test"` -> `BUILD SUCCESS` (76 testes).

---

## Fase 3: Camada de Aplicação (Ports, Use Cases, DTOs & Services)

### 1. Modificações Realizadas
- **Inbound Ports**:
  - [`CreateCharacterUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/CreateCharacterUseCase.java)
  - [`GetCharacterUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/GetCharacterUseCase.java)
  - [`ListCharactersUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ListCharactersUseCase.java)
  - [`UpdateCharacterUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/UpdateCharacterUseCase.java)
  - [`DeleteCharacterUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/DeleteCharacterUseCase.java)
  - [`RollAbilityScoresUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/RollAbilityScoresUseCase.java)
  - [`RollHitPointsUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/RollHitPointsUseCase.java)
- **Outbound Ports**:
  - [`PlayerCharacterRepositoryPort.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/PlayerCharacterRepositoryPort.java)
  - [`AbilityRollRepositoryPort.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/AbilityRollRepositoryPort.java)
  - [`HpRollRepositoryPort.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/HpRollRepositoryPort.java)
  - [`ReferenceDataRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java)
- **Application Services & Hydrator**:
  - [`CreateCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java)
  - [`CharacterQueryService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterQueryService.java)
  - [`UpdateCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/UpdateCharacterService.java)
  - [`DeleteCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/DeleteCharacterService.java)
  - [`RollService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RollService.java)
  - [`CharacterSheetHydrator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydrator.java)
- **Testes Unitários da Aplicação (Mockito)**:
  - [`CreateCharacterServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterServiceTest.java) (TP-01, TE-01)
  - [`RollServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RollServiceTest.java) (TP-02)
  - [`CharacterQueryServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterQueryServiceTest.java) (TP-03, TE-02)
  - [`UpdateAndDeleteCharacterServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/UpdateAndDeleteCharacterServiceTest.java) (TP-03, TE-02)

### 2. Resultados de Verificação
- `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.application.**.*Test"` -> `BUILD SUCCESS` (18 testes).

---

## Fase 4: Adaptador de Persistência, Entidades JPA & Testes de Integração

### 1. Modificações Realizadas
- **Entidades JPA & Mapeamento**:
  - [`PlayerCharacterJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java)
  - [`ClassProgressionJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ClassProgressionJpaEntity.java)
  - [`AbilityRollJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/AbilityRollJpaEntity.java) com `@JdbcTypeCode(SqlTypes.ARRAY)` e `@JdbcTypeCode(SqlTypes.JSON)`.
  - [`HpRollJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/HpRollJpaEntity.java) com `@JdbcTypeCode(SqlTypes.ARRAY)`.
- **Spring Data Repositories & Adaptadores**:
  - [`SpringDataPlayerCharacterRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataPlayerCharacterRepository.java)
  - [`SpringDataAbilityRollRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataAbilityRollRepository.java)
  - [`SpringDataHpRollRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataHpRollRepository.java)
  - [`SpringDataClassProgressionRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataClassProgressionRepository.java)
  - [`PlayerCharacterPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterPersistenceAdapter.java)
  - [`AbilityRollPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/AbilityRollPersistenceAdapter.java)
  - [`HpRollPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/HpRollPersistenceAdapter.java)
- **Testes de Integração com Testcontainers**:
  - [`PlayerCharacterPersistenceAdapterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterPersistenceAdapterTest.java) (TP-03, TE-02)
  - [`RollPersistenceAdapterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/RollPersistenceAdapterTest.java) (TP-02)

### 2. Resultados de Verificação
- `mvn -f player-character-service/backend/pom.xml test -Dtest="*Persistence*Test"` -> `BUILD SUCCESS` (6 testes).

---

## Fase 5: Adaptador Web, Endpoints REST, Validação & Testes MockMvc

### 1. Modificações Realizadas
- **Web DTOs com Validação Jakarta**:
  - [`CreateCharacterRequest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateCharacterRequest.java), [`UpdateCharacterRequest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/UpdateCharacterRequest.java), [`RollHpRequest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/RollHpRequest.java), [`CharacterSheetResponse.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterSheetResponse.java), [`CharacterSummaryResponse.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterSummaryResponse.java), etc.
- **Controller REST**:
  - [`PlayerCharacterController.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java) implementando `POST /ability-rolls`, `POST /hp-rolls`, `POST`, `GET`, `GET/{id}`, `PUT/{id}`, `DELETE/{id}` exigindo `X-User-Id`.
- **Tratamento de Exceções Global RFC 7807**:
  - [`GlobalExceptionHandler.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/core/web/GlobalExceptionHandler.java) (`404` para Not Found/IDOR, `422` para Validation/Tokens, `409` para conflito de rolls, `401` para header ausente).
- **Testes de API Web (MockMvc)**:
  - [`PlayerCharacterControllerTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterControllerTest.java) cobrindo TP-01, TP-02, TP-03, TE-01, TE-02, TE-03 e falta do header `X-User-Id`.

### 2. Resultados de Verificação
- `mvn -f player-character-service/backend/pom.xml test -Dtest="*PlayerCharacterController*Test"` -> `BUILD SUCCESS` (8 testes).

---

## Fase 6: Validação Completa, Delta Specs & Consolidação

### 1. Documentação Atualizada
- **Delta Spec de Domínio**:
  - [`player-character-domain-delta.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/specs/player-character-domain-delta.md): Documentados todos os contratos REST, regras de Ability Scores (`STANDARD_ARRAY`, `POINT_BUY`, `ROLLED`), regras de Dual Ruleset ASI (2014 linhagem vs 2024 antecedente), cálculo de HP server-side (`MAX`, `AVERAGE`, `ROLLED`), restrições de mutabilidade e isolamento de tenant via `X-User-Id`.
- **Delta Spec do Modelo de Dados**:
  - [`player-character-data-model-delta.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/specs/player-character-data-model-delta.md): Documentadas as colunas adicionadas a `characters` (`notes`, `hp_calculation_method`, `has_pending_level_choices`, `user_id NOT NULL`), a tabela `class_progressions` e as tabelas de rolagens com índices parciais únicos (`ability_rolls` e `hp_rolls`).
- **Checklist de Tarefas**:
  - [`tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/character-crud-backend/tasks.md): Todas as tarefas marcadas com `[x]`.

### 2. Quality Gate Final do Maven

```bash
mvn -f player-character-service/backend/pom.xml clean verify
```
- **Resultado:** `BUILD SUCCESS`
- **Métricas:** **119 testes executados, 0 falhas, 0 erros, 1 skipped intencional**.
- Todas as camadas (Domínio, Aplicação, Persistência, Web e Core) testadas rigorosamente cobrindo fluxos felizes e de exceção.

---

## Próximos Passos Recomendados

1. **Validação Final Pré-Arquivo**:
   - Executar `sdd-spec-validate` para conferência final dos critérios de aceitação.
2. **Arquivamento da Especificação**:
   - Executar a skill `sdd-spec-archive` para consolidar os deltas em `openspec/specs/` e arquivar `character-crud-backend`.
3. **Integração no Frontend**:
   - Conectar os novos endpoints do backend ao Wizard de Criação e Dashboard da Ficha de Personagem no `dungeoneer-frontend`.
