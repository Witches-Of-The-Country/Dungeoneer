# Tasks: Background Origin Feats Restriction, Ability Score Reroll Rules & Wizard Level Input Clamping

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Execution Order:** `004`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy Data & Schema Migration Tasks

- [x] Create Flyway migration `V32__align_origin_feats_and_reroll_permissions.sql` to align `player_character.backgrounds.origin_feat_name` with localized/canonical feat names, ensure only SRD 2024 Origin Feats have `feat_type = 'ORIGIN'`, and add `allow_ability_reroll BOOLEAN NOT NULL DEFAULT FALSE` to `player_character.characters`.

## Implementation Tasks

- [x] Add validation in `CreateCharacterService.java` rejecting any `originFeatId` whose `FeatType != FeatType.ORIGIN`.
- [x] Update `dungeoneer-frontend/app/characters/new/page.tsx` so the Background step strictly filters selectable feats to `feat.featType === 'ORIGIN'` and properly matches `selectedBackground.originFeatName`.
- [x] Refactor character level input in `dungeoneer-frontend/app/characters/new/page.tsx` (Step 3: Classe) to allow intermediate string editing without premature clamping during `onChange`, validating and clamping to `[1, 20]` on `onBlur`.
- [x] Update `PlayerCharacter.java`, `PlayerCharacterJpaEntity.java`, and `RollService.java` to support `allowAbilityReroll` and ability score rerolling with derived HP/AC recalculation.
- [x] Expose `POST /api/characters/{id}/reroll-ability-scores` and `PATCH /api/characters/{id}/reroll-permission` in `PlayerCharacterController.java`.
- [x] Update `dungeoneer-frontend/app/characters/[id]/edit/page.tsx` (and character sheet / campaign DM controls) to allow free rerolling when unbound (`!character.campaignId`) and DM-gated rerolling when bound to a campaign.

## Test Tasks

- [x] Add unit tests in `CreateCharacterServiceTest` verifying non-ORIGIN feats are rejected as `originFeatId`.
- [x] Add unit tests in `RollServiceTest` / `UpdateCharacterServiceTest` covering unbound reroll, blocked campaign reroll (`allowAbilityReroll = false`), and authorized campaign reroll (`allowAbilityReroll = true`).
- [x] Add frontend unit tests verifying Origin Feat filter in the creation wizard, level input editability (clearing and typing 20), and reroll button state.

## Documentation Tasks

- [x] Update delta specs in `specs/player-character-domain-delta.md`, `specs/player-character-data-model-delta.md`, and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`mvn clean test` and `npm test`)
- [x] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
