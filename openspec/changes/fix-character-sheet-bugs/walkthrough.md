# Walkthrough: Correção dos Bugs da Ficha de Personagem

> **Change ID:** `fix-character-sheet-bugs`
> **Status:** Concluído (Fases 1, 2, 3 e 4 Concluídas)

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

---

## Fase 3: Correção de Drag & Drop Lag e Guard de Rolagem de Vida (Frontend)

### O que foi implementado:
1. **Guard de Rolagem de HP no Wizard (`app/characters/new/page.tsx`)**:
   - Botão `hp-roll-button` desabilitado quando `!selectedClassId`.
   - Mensagem de aviso explicativa renderizada abaixo do botão: *"Selecione uma classe primeiro para determinar o dado de vida."*
2. **Eliminação de Latência no Drag & Drop de Atributos (`DraggableScoreChip`)**:
   - Remoção dinâmica da classe `transition-all` quando `isDragging` está ativo, garantindo acompanhamento em tempo real (< 16ms) do cursor do mouse sem arrasto retardado por transições CSS.
3. **Commit da Fase 3**:
   - Realizado no repositório `dungeoneer-frontend` com mensagem `fix(frontend): remove transition lag in DnD and guard HP rolling before class selection` (commit `d6cc597`).

---

## Fase 4: Traduções SRD PT-BR, Notas Incrementais e Suite de Testes

### O que foi implementado:
1. **Dicionário de Traduções SRD (`lib/srd-translations.ts`)**:
   - Mapas completos cobrindo:
     - 12 Classes SRD (Guerreiro, Mago, Ladino, Clérigo, Bárbaro, Bardo, Bruxo, Druida, Feiticeiro, Monge, Paladino, Patrulheiro).
     - 9 Linhagens/Raças SRD (Anão, Elfo, Humano, Halfling, Draconato, Gnomo, Meio-Elfo, Meio-Orc, Tiferino).
     - 10 Antecedentes SRD (Acólito, Artesão de Guilda, Charlatão, Criminoso, Eremita, Forasteiro, Herói do Povo, Marinheiro, Nobre, Sábio, Soldado).
     - 18 Perícias SRD (Acrobacia, Atletismo, Percepção, Furtividade, etc.).
     - Mais de 40 Magias e Talentos de Origem SRD 2024.
   - Função utilitária `translateReferenceName(name)` com fallback seguro para texto original quando sem tradução cadastrada.
2. **Integração nas Camadas de API e Consulta (`lib/api/characters.ts`)**:
   - Tradução transparente de respostas das queries de classes, linhagens, antecedentes e fichas de personagens.
   - Definição dos tipos `CharacterNote` e `CreateCharacterNotePayload`.
   - Hooks TanStack Query `useCharacterNotes(characterId)` e `useAddCharacterNote(characterId)`.
3. **Componentes de UI de Notas Incrementais**:
   - `components/note-card.tsx`: Card temático Dark Fantasy exibindo título da nota, data formatada em PT-BR (`pt-BR`) e corpo com formatação preservada.
   - `components/add-note-form.tsx`: Formulário com validação Zod (título obrigatório até 255 chars, corpo obrigatório até 10.000 chars) e mutação automática com feedback Sonner.
4. **Refatoração da Ficha de Personagem e Página de Edição**:
   - `app/characters/[id]/page.tsx`: Aba "Notas" atualizada com formulário `<AddNoteForm>` e listagem cronológica `<NoteCard>`, com estados de loading e lista vazia.
   - `app/characters/[id]/edit/page.tsx`: Remoção do campo monolítico de notas legado.
5. **Cobertura de Testes Automatizados**:
   - `__tests__/wizard-ability-roll-guard.test.tsx`: Validação do botão desabilitado sem classe e habilitado após seleção de classe.
   - `__tests__/character-sheet.test.tsx`: Teste da aba "Notas" incremental com adição e renderização via mocks da API.
   - `__tests__/character-list-and-edit.test.tsx`: Verificação da listagem e formulário de edição atualizado.
   - `__tests__/incremental-notes-and-translations.test.tsx`: Suite dedicada para `translateReferenceName`, `NoteCard` e validação de schema do `AddNoteForm`.

### Resultados Finais de Verificação:
```bash
# Frontend Unit Tests (Vitest)
Test Files  7 passed (7)
Tests       15 passed (15)

# Frontend Production Build (Next.js Turbopack)
✓ Compiled successfully in 49s
✓ Generating static pages (9/9) in 2.2s

# Backend Unit & Integration Tests (Maven Surefire + Testcontainers)
Tests run: 146, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```
