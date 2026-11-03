# Tasks: Add Homebrew Feat Charges

> **Change ID:** `add-homebrew-feat-charges`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Add `ChargeType`/`ChargeRecoveryType` enums and fields to `MechanicalEffects`
- [x] Add pure charge calculator (max + recovery) in domain
- [x] Update `CharacterFeat.recoverCharges` (short/long independent, clamping)
- [x] Update `CharacterSheetHydrator`, `CreateCharacterService`, `LevelUpService`
- [x] Update DTOs, `HomebrewService` validation, JPA JSONB compat
- [x] Frontend: charge builder in `create-homebrew-dialog.tsx`, types, `feats-tab.tsx`

## Test Tasks

- [x] Unit tests for calculator and `CharacterFeat` (TP-01/02, TE-01/02/05)
- [x] MockMvc/Integration tests (TP-03, TE-03/04)
- [x] Vitest for dialog and feats tab (TP-04)

## Documentation Tasks

- [x] Fill delta specs (domain, data-model, frontend)

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
