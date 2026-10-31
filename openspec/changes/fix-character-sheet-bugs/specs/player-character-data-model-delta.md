# Delta Spec: `player-character-data-model`

> **Change ID:** `fix-character-sheet-bugs`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Nova tabela: `character_note`

```sql
CREATE TABLE character_note (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  character_id UUID        NOT NULL REFERENCES player_character(id) ON DELETE CASCADE,
  title        VARCHAR(255) NOT NULL,
  body         TEXT        NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_character_note_character_id ON character_note(character_id);
```

### Deprecação: campo `notes TEXT` em `player_character`

O campo `notes TEXT` na tabela `player_character` deve ser marcado como deprecated no schema e removido após migração dos dados para `character_note`.

### Migration Flyway a criar

- `V<N>__create_character_note_table.sql` — cria tabela `character_note`
- `V<N+1>__migrate_notes_to_character_note.sql` — copia conteúdo existente de `notes` para primeira nota com título "Nota importada"
- `V<N+2>__drop_notes_column.sql` — remove coluna `notes` de `player_character` (executar apenas após validação em produção)
