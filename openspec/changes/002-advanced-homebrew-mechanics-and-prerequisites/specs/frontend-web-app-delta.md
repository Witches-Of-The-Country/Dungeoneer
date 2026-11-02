# Delta Spec: `frontend-web-app`

> **Change ID:** `002-advanced-homebrew-mechanics-and-prerequisites`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Enhanced `CreateHomebrewDialog` Forms
- **Weapons**: `+ Adicionar Dado de Dano` list with per-roll `Condicional` checkbox and condition note.
- **Feats**: Fixed or `"Escolha N entre [...]"` ability bonus mode, `+ Adicionar Pré-requisito` rule builder referencing DB entities, and `Condicional` toggle on passive bonuses.
- **Spells**: `V`, `S`, `M` checkboxes, numeric + unit dropdowns for Casting Time, Range, and Duration (`Instantânea` toggle), plus incremental damage/effect rows.
- **Classes**: Interactive 6-ability multi-select for Saving Throws and `Tipo de Preparação de Magias` (`Preparadas`, `Conhecidas`, `Nenhuma`).
- **Lineages**: Inline `+ Adicionar Característica de Linhagem` builder.
