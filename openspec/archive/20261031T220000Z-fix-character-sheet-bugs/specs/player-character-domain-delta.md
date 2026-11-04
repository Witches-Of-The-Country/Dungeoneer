# Delta Spec: `player-character-domain`

> **Change ID:** `fix-character-sheet-bugs`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Entidade de Domínio `CharacterNote`

Modelo de domínio puro sem dependências de framework:
- `id: UUID`
- `characterId: UUID`
- `title: String` (invariante: não nulo, não vazio, tamanho $\le 255$)
- `body: String` (invariante: não nulo, não vazio, tamanho $\le 10{,}000$)
- `createdAt: Instant`

Validações de domínio lançam `CharacterValidationException` caso invariantes sejam violadas.

### 2. Inbound Ports (Use Cases)

```java
public interface AddCharacterNoteUseCase {
    CharacterNoteResult addNote(UUID characterId, UUID userId, AddCharacterNoteCommand command);
}

public interface ListCharacterNotesUseCase {
    List<CharacterNoteResult> listNotes(UUID characterId, UUID userId);
}
```

### 3. Outbound Port (Repository)

```java
public interface CharacterNoteRepositoryPort {
    CharacterNote save(CharacterNote note);
    List<CharacterNote> findAllByCharacterIdOrderByCreatedAtDesc(UUID characterId);
}
```

### 4. REST Endpoints

- `POST /api/characters/{id}/notes` — Cria uma nota vinculada ao personagem (requer `X-User-Id` para validação de posse, retorna 201 Created).
- `GET /api/characters/{id}/notes` — Retorna lista cronológica decrescente das notas do personagem (retorna 200 OK).

### 5. Validação em `RollHitPointsUseCase`

- `RollHitPointsCommand.archetypeId()` deve ser explicitamente validado como não-nulo antes de consultar repositórios.
- Se nulo ou se o arquétipo não for encontrado, lança `CharacterValidationException` mapeado para RFC 7807 `422 Unprocessable Entity` / `400 Bad Request`.
