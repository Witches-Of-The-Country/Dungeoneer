# Delta Spec: `player-character-domain`

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### MODIFIED: `UC4 — Rest & Resource Recovery (Feat Charges)`
- `FeatChargeCalculator` and `RestService` SHALL dynamically recalculate `maxCharges` and `recoveryType` for character feats whose persisted `maxCharges` is `null` (legacy feats such as Rage/Fúria).
- Short rest and long rest recovery SHALL respect ruleset-specific recovery rules (e.g., `SRD_2024` Rage recovers 1 charge on a short rest and all charges on a long rest; `SRD_2014` Rage recovers all charges on a long rest).
- Flyway migration `V28__adapt_legacy_feat_charges_and_recovery.sql` backfills legacy feat charge metadata.

### MODIFIED: `UC-HB — Homebrew Content Mutation Ownership Enforcement`
- All `PUT /api/reference/{spells,feats,items,classes,lineages,subclasses}/{id}` endpoints in `ReferenceDataController` SHALL require a non-null `X-User-Id` header (`required = true`).
- `HomebrewService` update methods SHALL verify both `existing.getRuleset() == Ruleset.HOMEBREW` AND `existing.getOwnerId() != null && existing.getOwnerId().equals(userId)`. If the caller is not the owner, the service SHALL throw an authorization exception mapped to HTTP `403 Forbidden` (RFC 7807 `ProblemDetail`).

### MODIFIED: `Error Handling (RFC 7807 ProblemDetail)`
- `GlobalExceptionHandler.handleGenericException(Exception)` SHALL log the underlying exception internally and return a generic safe message (`"An unexpected internal error occurred."`) in the `ProblemDetail` response to prevent leaking internal SQL or stack trace details.
