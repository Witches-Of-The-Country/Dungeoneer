# Handoff Document — Dungeoneer: `fix-character-sheet-bugs`

> **Documento de Transição para Agente com Contexto Limpo**  
> **Data:** 2026-10-30  
> **Change ID:** `fix-character-sheet-bugs`  
> **Status:** Fases 1, 2 e 3 implementadas e commitadas. Próximo passo: Fase 4 (Frontend SRD Translations & Notas UI).

---

## 1. Visão Geral da Mudança

Este change aborda 4 correções cruciais identificadas na experiência de criação e visualização da ficha de personagem:
1. **Bug 1 — Rolagem de vida após escolha de classe**: Bloqueio da rolagem no wizard (frontend) e no `RollService` (backend) enquanto `class` for nulo.
2. **Bug 2 — Conteúdo da SRD em PT-BR**: Dicionários de tradução em `lib/srd-translations.ts` aplicados nas consultas TanStack Query para classes, raças, antecedentes, talentos e perícias.
3. **Bug 3 — Notas incrementais**: Refatoração do campo de notas monolítico para registros independentes com `title`, `body` e timestamp, com tabela dedicada e endpoints REST.
4. **Bug 4 — Lag no Drag & Drop**: Eliminação da latência do cursor provocada pela interpolação de `transition-all` CSS no `DraggableScoreChip` durante o arraste.

---

## 2. Estado da Implementação por Bug

| Bug | Camada | Status | Detalhes & Arquivos |
|---|---|---|---|
| **Bug 1** (HP Roll Guard) | **Backend** | ✅ **Concluído** | Validação em `RollService.java` rejeitando `archetypeId == null`. Testes unitários em `RollServiceTest` e integração MockMvc (422) em `PlayerCharacterControllerTest`. |
| **Bug 1** (HP Roll Guard) | **Frontend** | ✅ **Concluído** | Em `app/characters/new/page.tsx`: botão de rolagem de HP desabilitado sem classe selecionada, remoção de fallback `classes[0]`, mensagem explicativa `"Selecione uma classe primeiro"`. Teste adicionado em `wizard-ability-roll-guard.test.tsx`. |
| **Bug 2** (Tradução SRD) | **Frontend** | ⏳ **Pendente (Fase 4)** | Arquivo `dungeoneer-frontend/lib/srd-translations.ts` e integração com `lib/api/characters.ts` e páginas de listagem e ficha. |
| **Bug 3** (Notas Incrementais) | **Backend** | ✅ **Concluído 100%** | Flyway `V5__create_character_note_table.sql`, entidade pura `CharacterNote.java`, ports `AddCharacterNoteUseCase`, `ListCharacterNotesUseCase`, `CharacterNoteRepositoryPort`, `CharacterNoteService`, DTOs Jakarta (`CreateCharacterNoteRequest`, `CharacterNoteResponse`), endpoints `POST /api/characters/{id}/notes` (201) e `GET /api/characters/{id}/notes` (200). 146 testes passando no backend! |
| **Bug 3** (Notas Incrementais) | **Frontend** | ⏳ **Pendente (Fase 4)** | Criar `NoteCard.tsx`, `AddNoteForm.tsx`, hooks no `characters.ts` e substituir a textarea única em `app/characters/[id]/page.tsx`. |
| **Bug 4** (Lag no DnD) | **Frontend** | ✅ **Concluído** | Removida classe `transition-all` durante `isDragging` em `DraggableScoreChip` em `app/characters/new/page.tsx`. Arraste com resposta direta aos eventos do ponteiro (< 16ms). |

---

## 3. Estado do Git & Commits Realizados

O projeto utiliza **git submodules** para os serviços e frontend. Os commits foram realizados separadamente em cada repositório:

### 1. Submódulo `player-character-service`
- **Branch:** `feat/character-creation`
- **Commit:** `ea32761`
- **Mensagem:** `feat(backend): add CharacterNote domain model, V5 Flyway migration, use cases and REST endpoints`
- **Arquivos commitados:**
  - `backend/src/main/resources/db/migration/V5__create_character_note_table.sql`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterNote.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java` (deprecação de `notes`)
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/CharacterNoteRepositoryPort.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/AddCharacterNoteUseCase.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ListCharacterNotesUseCase.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/AddCharacterNoteCommand.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterNoteResult.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteService.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RollService.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateCharacterNoteRequest.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterNoteResponse.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/PlayerCharacterWebMapper.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNoteJpaEntity.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataCharacterNoteRepository.java`
  - `backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapter.java`
  - `backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/CharacterNoteTest.java`
  - `backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteServiceTest.java`
  - `backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RollServiceTest.java`
  - `backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapterTest.java`
  - `backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterControllerTest.java`

### 2. Submódulo `dungeoneer-frontend`
- **Branch:** `feature/login-page`
- **Commit:** `d6cc597`
- **Mensagem:** `fix(frontend): guard wizard HP roll and eliminate drag-and-drop latency`
- **Arquivos commitados:**
  - `app/characters/new/page.tsx`
  - `__tests__/wizard-ability-roll-guard.test.tsx`

### 3. Repositório Raiz (`Dungeoneer`)
- **Branch:** `dev`
- **Arquivos preparados para commit:**
  - `openspec/changes/fix-character-sheet-bugs/proposal.md`
  - `openspec/changes/fix-character-sheet-bugs/tasks.md`
  - `openspec/changes/fix-character-sheet-bugs/walkthrough.md`
  - `openspec/changes/fix-character-sheet-bugs/specs/player-character-data-model-delta.md`
  - `openspec/changes/fix-character-sheet-bugs/specs/player-character-domain-delta.md`
  - `openspec/changes/fix-character-sheet-bugs/specs/frontend-web-app-delta.md`
  - `prompt-fix-character-sheet-bugs.md`
  - `HANDOFF-FIX-CHARACTER-SHEET-BUGS.md`
  - Ponteiros atualizados dos submódulos `dungeoneer-frontend` e `player-character-service`.

---

## 4. Roteiro Passo a Passo para o Próximo Agente (Fase 4)

O próximo agente deve focar na **Fase 4: Frontend SRD Translations & Notas UI**.

### Passo 1: Implementar `srd-translations.ts`
Criar `dungeoneer-frontend/lib/srd-translations.ts`:
- Dicionários com mapeamentos de classes, linhagens, antecedentes, magias e perícias para português (`pt-BR`).
- Função `translateSrd(key: string, map: Record<string, string>): string` com fallback para a chave original.
- Mapeamentos principais:
  - **Classes**: Barbarian -> Bárbaro, Bard -> Bardo, Cleric -> Clérigo, Druid -> Druida, Fighter -> Guerreiro, Monk -> Monge, Paladin -> Paladino, Ranger -> Guardião, Rogue -> Ladino, Sorcerer -> Feiticeiro, Warlock -> Bruxo, Wizard -> Mago.
  - **Lineages**: Human -> Humano, Elf -> Elfo, Dwarf -> Anão, Halfling -> Halfling, Dragonborn -> Draconato, Gnome -> Gnomo, Half-Elf -> Meio-Elfo, Half-Orc -> Meio-Orc, Tiefling -> Tiefling.
  - **Backgrounds**: Acolyte -> Acólito, Criminal -> Criminoso, Folk Hero -> Herói do Povo, Noble -> Nobre, Sage -> Sábio, Soldier -> Soldado.

### Passo 2: Atualizar `dungeoneer-frontend/lib/api/characters.ts`
- Adicionar interface:
  ```ts
  export interface CharacterNote {
    id: string;
    characterId: string;
    title: string;
    body: string;
    createdAt: string;
  }
  ```
- Adicionar query hook `useCharacterNotes(characterId: string)`: `GET /api/characters/${characterId}/notes`
- Adicionar mutation hook `useAddCharacterNote(characterId: string)`: `POST /api/characters/${characterId}/notes` com invalidação da query `['character-notes', characterId]`.
- Aplicar `translateSrd` nos seletores de `useCharacters`, `useReferenceClasses`, `useReferenceLineages` e `useReferenceBackgrounds`.

### Passo 3: Criar Componentes de Notas no Frontend
- `dungeoneer-frontend/components/note-card.tsx`: exibe `title`, `body` e data formatada com `new Date(createdAt).toLocaleDateString('pt-BR')`.
- `dungeoneer-frontend/components/add-note-form.tsx`: formulário com título e corpo, validação e botão de salvar.
- Atualizar a aba de notas em `dungeoneer-frontend/app/characters/[id]/page.tsx` para exibir o formulário e a lista cronológica reversa de notas.

### Passo 4: Testes e Finalização
- Rodar e atualizar `dungeoneer-frontend/__tests__/character-sheet.test.tsx` e `character-list-and-edit.test.tsx`.
- Executar `sdd-spec-archive` para consolidar o change proposal `fix-character-sheet-bugs`.
