# Walkthrough: Correção dos Bugs da Ficha de Personagem

> **Change ID:** `fix-character-sheet-bugs`
> **Status:** Em Progresso (Fase 1 e Fase 2 Concluídas)

---

## Fase 1: Entidades de Domínio, Migração Flyway V5 e Guard de Rolagem de HP

### O que foi implementado:
1. **Migração Flyway `V5__create_character_note_table.sql`**:
   - Criação da tabela `player_character.character_notes` com `id`, `character_id`, `title`, `body` e `created_at`.
   - Criação do índice `idx_character_notes_character_id`.
   - Migração automática de dados legados do campo `notes` para o primeiro registro com título `'Nota Inicial'`.
2. **Entidade de Domínio Pura `CharacterNote`**:
   - Criada em `domain/model/CharacterNote.java` sem dependências de framework.
   - Validações de invariantes: título obrigatório (máximo 255 caracteres) e corpo obrigatório (máximo 10.000 caracteres), lançando `CharacterValidationException`.
   - Deprecação documentada do campo e getter `notes` em `PlayerCharacter.java`.
3. **Porta de Saída e Adaptador de Persistência**:
   - Porta `CharacterNoteRepositoryPort` em `application/port/out/`.
   - Entidade JPA `CharacterNoteJpaEntity` em `adapter/out/persistence/`.
   - Interface `SpringDataCharacterNoteRepository`.
   - Adaptador `CharacterNotePersistenceAdapter`.
4. **Guard de Validação no Backend para Rolagem de HP**:
   - Em `RollService.java`, adicionada validação explícita de `archetypeId != null`. Se nulo ou ausente, lança `CharacterValidationException`.
5. **Testes Unitários**:
   - `CharacterNoteTest`: 8 testes cobrindo validações de tamanho, nulidade, trim e igualdade.
   - `RollServiceTest`: Novos testes verificando rejeição de `archetypeId` nulo, `userId` nulo e arquétipo inexistente.
   - `CharacterNotePersistenceAdapterTest`: Testes de conversão e ordenação cronológica.

### Resultados dos Testes:
```bash
Tests run: 135, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```

---

### Comando de Commit da Fase 1:
```bash
git add player-character-service/backend/src/main/resources/db/migration/V5__create_character_note_table.sql \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterNote.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/CharacterNoteRepositoryPort.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNoteJpaEntity.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataCharacterNoteRepository.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapter.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RollService.java \
        player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/CharacterNoteTest.java \
        player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RollServiceTest.java \
        player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapterTest.java
git commit -m "feat(backend): add CharacterNote domain model, V5 Flyway migration, and HP roll validation"
```

---

## Fase 2: Casos de Uso, Endpoints REST e Testes de Integração de API

### O que foi implementado:
1. **Ports de Entrada e DTOs de Aplicação**:
   - `AddCharacterNoteUseCase` e `ListCharacterNotesUseCase` em `application/port/in/`.
   - `AddCharacterNoteCommand` e `CharacterNoteResult` em `application/dto/`.
2. **Serviço de Aplicação `CharacterNoteService`**:
   - Implementa adição e listagem de notas com validação rigorosa de posse do personagem (`playerCharacterRepository.findByIdAndUserId`).
   - Lança `CharacterNotFoundException` se o personagem não pertencer ao usuário ou não existir.
3. **Adaptador Web e Endpoints REST**:
   - DTOs `CreateCharacterNoteRequest` (com Jakarta `@NotBlank`, `@Size(max=255)` e `@Size(max=10000)`) e `CharacterNoteResponse`.
   - Métodos no `PlayerCharacterWebMapper` para converter DTOs da API e de domínio.
   - Endpoints no `PlayerCharacterController`:
     - `POST /api/characters/{id}/notes` -> Retorna `201 Created` com header `Location` e corpo `CharacterNoteResponse`.
     - `GET /api/characters/{id}/notes` -> Retorna `200 OK` com `List<CharacterNoteResponse>`.
4. **Testes Unitários e de Integração**:
   - `CharacterNoteServiceTest`: 6 testes cobrindo fluxo feliz de adição, listagem e casos de erro/ownership.
   - `PlayerCharacterControllerTest`: Novos testes MockMvc validando 201 Created no POST, 200 OK no GET, 422 Unprocessable Entity para título vazio/corpo excessivo e 422 para rolagem de HP sem arquétipo.

### Resultados dos Testes:
```bash
Tests run: 146, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```

---

### Comando de Commit da Fase 2:
```bash
git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/AddCharacterNoteUseCase.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ListCharacterNotesUseCase.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/AddCharacterNoteCommand.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterNoteResult.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteService.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateCharacterNoteRequest.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterNoteResponse.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/PlayerCharacterWebMapper.java \
        player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java \
        player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteServiceTest.java \
        player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterControllerTest.java
git commit -m "feat(backend): add REST endpoints and use cases for incremental character notes"
```
