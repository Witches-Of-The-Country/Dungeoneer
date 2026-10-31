# Delta Spec: `player-character-data-model`

> **Change ID:** `refactor-wizard-hp-srd-and-session`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Flyway Migration V6: Adição de `skill_proficiencies`
- Tabela `player_character.characters`:
```sql
ALTER TABLE player_character.characters
    ADD COLUMN IF NOT EXISTS skill_proficiencies VARCHAR[] NOT NULL DEFAULT '{}';
```
- Armazena o array de strings com todas as perícias dominadas pelo personagem (combinação das perícias de classe selecionadas e perícias do antecedente).

### 2. Tabela `class_progressions`
- Reutiliza a tabela existente criada na Migration V4:
```sql
CREATE TABLE IF NOT EXISTS player_character.class_progressions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    character_id UUID NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE,
    archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id),
    subclass_id UUID NULL,
    class_level INT NOT NULL CHECK (class_level >= 1 AND class_level <= 20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_class_progression_character_archetype UNIQUE (character_id, archetype_id)
);
```
- Permite persistir múltiplos registros por personagem na criação quando houver multiclasse.
