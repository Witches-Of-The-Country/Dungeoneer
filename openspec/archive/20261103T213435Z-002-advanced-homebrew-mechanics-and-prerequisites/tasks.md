# Tasks: 002 — Advanced Homebrew Mechanics, Conditional Bonuses & Logical Prerequisite Builder

> **Change ID:** `002-advanced-homebrew-mechanics-and-prerequisites`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Create Flyway migration `V19__add_spell_mechanics_archetype_prep_and_lineage_feats.sql` adding `spells.mechanical_properties JSONB`, `archetypes.spell_preparation_type VARCHAR(20)`, and `lineage_feats` join table.
- [x] Update `DamageRoll.java`, `MechanicalEffects.java`, `Spell.java`, and `Archetype.java` to support conditional damage rolls, choice-based ability bonuses (`chooseCount`, `allowedAbilities`, `bonus`), structured hard `prerequisites`, conditional mechanical bonuses, spell damage/effects, and `spellPreparationType`.
- [x] Update `HomebrewService.java`, `CreateCharacterService.java`, and `LevelUpService.java` to persist inline lineage feats, validate structured feat prerequisites against character ability scores/class/lineage/feats/level, and apply chosen ability bonuses.
- [x] Update `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` and `lib/api/homebrew.ts` with:
  - Incremental `+ Adicionar Dado de Dano` and `Condicional` checkbox on Weapons.
  - `"Escolha N entre [Atributos]"` ability bonus builder, `+ Adicionar Pré-requisito` logical dependency builder, and `Condicional` toggle on Feats.
  - `V / S / M` checkboxes, numeric + unit dropdowns for Casting Time, Range, and Duration (with `"Instantânea"` toggle), and incremental `+ Adicionar Dado de Dano / Efeito` on Spells.
  - Multi-select Saving Throw chips and `spellPreparationType` (`PREPARED` / `KNOWN` / `NONE`) on Classes.
  - Inline `+ Adicionar Traço / Feat` builder on Lineages.

## Test Tasks

- [x] Add backend unit tests in `HomebrewServiceTest.java` and `LevelUpServiceTest.java` covering conditional weapon damage rolls, choice ability bonuses, structured feat prerequisites validation, spell mechanics, and inline lineage feats.
- [x] Add frontend Vitest tests in `__tests__/homebrew-creation.test.tsx` verifying the incremental weapon damage builder, feat prerequisite builder, spell V/S/M & unit inputs, and class multi-select saving throws.

## Documentation Tasks

- [x] Document the expanded mechanics schemas in `specs/player-character-domain-delta.md` and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
