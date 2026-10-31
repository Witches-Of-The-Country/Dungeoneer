# Delta Spec: `player-character-data-model`

> **Change ID:** `fix-character-sheet-bugs`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Nova tabela: `character_notes`

```sql
CREATE TABLE player_character.character_notes (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  character_id UUID        NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE,
  title        VARCHAR(255) NOT NULL,
  body         TEXT        NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_character_notes_character_id ON player_character.character_notes(character_id);
```

### Deprecação: campo `notes TEXT` em `player_character.characters`

O campo `notes TEXT` na tabela `player_character.characters` permanece para compatibilidade retrógrada e é populado na migração inicial.

### Migration Flyway a criar

- `V5__create_character_note_table.sql` — cria tabela `character_notes` no schema `player_character` e copia conteúdo existente de `notes` para a primeira nota (`title = 'Nota Inicial'`) onde `notes IS NOT NULL AND TRIM(notes) != ''`.
