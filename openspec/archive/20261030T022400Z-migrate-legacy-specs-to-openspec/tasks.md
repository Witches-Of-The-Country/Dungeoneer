# Tasks: Migrate Legacy Specs To Openspec

> **Change ID:** `migrate-legacy-specs-to-openspec`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Populate `openspec/changes/migrate-legacy-specs-to-openspec/specs/sdd-stack-contract-delta.md` with Dungeoneer stack details
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/system-architecture.md` from `SPEC.md`, `architecture-diagram.md`, and `ARCHITECTURE.md`
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/player-character-domain.md` from `README.md`, `USE-CASES.md`, `SPEC.md`, and `api-contracts.md`
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/player-character-data-model.md` from `data-models.md` and `er-diagram.md`
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/campaign-domain.md` from `SPEC.md`, `USE-CASES.md`, `er-diagram.md`, and `api-contracts.md`
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/frontend-web-app.md` from `frontend-spec.md` and `STATE.md`
- [x] Create `openspec/changes/migrate-legacy-specs-to-openspec/specs/infrastructure-and-testing.md` from `infrastructure.md` and `testing-strategy.md`
- [x] Update `AGENTS.md` and `README.md` (and any `.agents/skills/` references if applicable) to point to `openspec/specs/*.md`
- [x] Remove legacy files in `docs/` (`docs/spec/`, `docs/USE-CASES.md`, `docs/ARCHITECTURE.md`, `docs/STATE.md`) and `openspec/changes/_example/`

## Security Validation Tasks

- [x] Confirm no secrets or credentials are hardcoded in any migrated specification or `sdd-stack-contract.md`
- [x] Confirm Jakarta Validation and RFC 7807 security constraints are preserved in `player-character-domain.md`

## Shift-Left Test Suite Tasks

### Happy Path
- [x] Verify all canonical spec files in `openspec/changes/migrate-legacy-specs-to-openspec/specs/` contain valid YAML frontmatter and complete sections (TP-01)
- [x] Verify all links in `AGENTS.md` resolve to the migrated `openspec/specs/` paths (TP-02)

### Edge Cases & Failure Modes
- [x] Verify zero dangling references to `docs/spec/` remain in `AGENTS.md` and `.agents/` (TE-01)
- [x] Run backend test suite to verify zero regressions (TE-02)

## Documentation Tasks

- [x] Update `AGENTS.md` Specifications & Knowledge Index table
- [x] Consolidate specs into `openspec/specs/` and archive `migrate-legacy-specs-to-openspec` via `sdd-spec-archive`

## Review Gate

- [x] All implementation tasks complete
- [x] All security validation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [x] Ready for `sdd-spec-archive`
