# Delta Spec: `player-character-domain`

> **Change ID:** `refactor-wizard-hp-srd-and-session`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Atualização e Edição de Notas (`CharacterNote`)
- Nova porta de entrada `UpdateCharacterNoteUseCase` em `application/port/in/`:
  - `CharacterNoteResult updateNote(UUID characterId, UUID noteId, UUID userId, UpdateCharacterNoteCommand command)`
- Entidade de domínio `CharacterNote`:
  - Adição de métodos de negócio para mutação de título e corpo validando invariantes (título não-nulo até 255 caracteres, corpo não-nulo até 10.000 caracteres).
- Endpoint REST:
  - `PUT /api/characters/{id}/notes/{noteId}` com `@Valid UpdateCharacterNoteRequest` retornando `200 OK` e `CharacterNoteResponse`.

### 2. Criação com Suporte a Multiclasse e Perícias de Classe
- `CreateCharacterCommand` passa a suportar:
  - `List<ClassProgressionCommand> classProgressions` (para personagens criados em nível > 1 com distribuição multiclasse).
  - `List<String> skillProficiencies` (consolidação das perícias de classe selecionadas e perícias do antecedente).
- `CreateCharacterUseCase` persiste todas as progressões de classe e valida que a soma dos níveis das classes é exatamente igual ao nível total do personagem.
