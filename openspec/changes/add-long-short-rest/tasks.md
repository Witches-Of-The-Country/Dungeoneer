# Tasks: Add Long Short Rest

> **Change ID:** `add-long-short-rest`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [ ] Flyway `V27` (exhaustion, hit_dice_spent) + JPA mappings
- [ ] Domain: `PlayerCharacter.exhaustion`, `ClassProgression` Hit Dice methods
- [ ] `RulesEngine` rest methods + 2014/2024 implementations
- [ ] Rewrite `RestService` (short/long, temp HP fix, no-resurrection guard)
- [ ] Update `RestCommand`, `RestUseCase`, controller, sheet DTOs
- [ ] Frontend: API types, header buttons, `rest-modal.tsx`, Hit Dice counter, exhaustion badge

## Test Tasks

- [ ] Unit tests (TP-01/02, TE-01/03/05/06/07)
- [ ] MockMvc tests (TP-03, TE-02/03/04)
- [ ] Testcontainers integration (TP-04)
- [ ] Vitest (TP-05)

## Documentation Tasks

- [ ] Fill delta specs (domain, data-model, frontend)

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
