# Tasks: Add Long Short Rest

> **Change ID:** `add-long-short-rest`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Flyway `V27` (exhaustion, hit_dice_spent) + JPA mappings
- [x] Domain: `PlayerCharacter.exhaustion`, `ClassProgression` Hit Dice methods
- [x] `RulesEngine` rest methods + 2014/2024 implementations
- [x] Rewrite `RestService` (short/long, temp HP fix, no-resurrection guard)
- [x] Update `RestCommand`, `RestUseCase`, controller, sheet DTOs
- [x] Frontend: API types, header buttons, `rest-modal.tsx`, Hit Dice counter, exhaustion badge

## Test Tasks

- [x] Unit tests (TP-01/02, TE-01/03/05/06/07)
- [x] MockMvc tests (TP-03, TE-02/03/04)
- [x] Testcontainers integration (TP-04)
- [x] Vitest (TP-05)

## Documentation Tasks

- [x] Fill delta specs (domain, data-model, frontend)

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
