# Walkthrough — Implementação do Character CRUD Backend (`character-crud-backend`)

Este documento registra o progresso cumulativo das fases de implementação da especificação `character-crud-backend` no serviço `player-character-service`.

---

## Fase 1: Database Migration & Remoção de Stubs Inativos

### 1. Modificações Realizadas

- **Migração Flyway criada**:
  - [`V4__add_character_crud_and_roll_tables.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V4__add_character_crud_and_roll_tables.sql)
    - Adicionadas colunas à tabela `player_character.characters`:
      - `notes TEXT NOT NULL DEFAULT ''`
      - `hp_calculation_method VARCHAR(15) NOT NULL DEFAULT 'MAX'`
      - `has_pending_level_choices BOOLEAN NOT NULL DEFAULT FALSE`
      - Imposição de restrição `user_id UUID NOT NULL` (com atualização prévia de eventuais registros nulos)
      - Índice `idx_characters_user_id` em `characters(user_id)`
    - Criada a tabela `player_character.class_progressions` com chave estrangeira em cascata para `characters(id)` e `archetypes(id)`, restrição de unicidade `(character_id, archetype_id)` e índice `idx_class_progressions_character_id`.
    - Criada a tabela `player_character.ability_rolls` com array de inteiros `rolled_scores`, breakdown em JSONB `dice_breakdown` e índice parcial único `idx_unused_ability_roll` em `(user_id) WHERE used = FALSE`.
    - Criada a tabela `player_character.hp_rolls` com `archetype_id`, `target_level` (2–20), `rolled_hp_per_level INT[]` e índice parcial único `idx_unused_hp_roll` em `(user_id, archetype_id, target_level) WHERE used = FALSE`.
- **Limpeza de Stubs Inativos**:
  - Removidos stubs órfãos e sem referências:
    - `CreatePlayerCharacterUseCase.java`
    - `CharacterRepositoryPort.java`
    - `CreatePlayerCharacterService.java`
    - `PlayerCharacterUseCaseConfig.java`

### 2. Resultados de Verificação & Quality Gate

- Execução com Maven global: `mvn -f player-character-service/backend/pom.xml test`
- **Resultado:**
  - `Tests run: 78, Failures: 0, Errors: 0, Skipped: 1`
  - `BUILD SUCCESS` (tempo total: ~47s, incluindo inicialização do Testcontainers PostgreSQL)

### 3. Checklist de Tarefas Concluídas

- [x] Migração Flyway `V4__add_character_crud_and_roll_tables.sql`
- [x] Remoção de stubs inativos sem referências

---

## Próximo Passo: Fase 2 — Camada de Domínio Pura & Testes Unitários (TDD)
Avançar para a implementação de `ClassProgression`, `AbilityScoreValidator`, `RulesEngine` (`Srd2014RulesEngine`, `Srd2024RulesEngine`), atualização do aggregate `PlayerCharacter` e testes unitários com JUnit 5.
