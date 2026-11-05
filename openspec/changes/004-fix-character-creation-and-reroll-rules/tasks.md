# Tasks: Background Origin Feats Restriction & Ability Score Reroll Rules

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Execution Order:** `004`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy Data & Schema Migration Tasks

- [ ] Create Flyway migration `V31__align_origin_feats_and_reroll_permissions.sql` to align `player_character.backgrounds.origin_feat_name` with localized/canonical feat names, ensure only SRD 2024 Origin Feats have `feat_type = 'ORIGIN'`, and add `allow_ability_reroll BOOLEAN NOT NULL DEFAULT FALSE` to `player_character.characters`.

## Implementation Tasks

- [ ] Add validation in `CreateCharacterService.java` rejecting any `originFeatId` whose `FeatType != FeatType.ORIGIN`.
- [ ] Update `dungeoneer-frontend/app/characters/new/page.tsx` so the Background step strictly filters selectable feats to `feat.featType === 'ORIGIN'` and properly matches `selectedBackground.originFeatName`.
- [ ] Update `PlayerCharacter.java`, `PlayerCharacterJpaEntity.java`, and `RollService.java` to support `allowAbilityReroll` and ability score rerolling with derived HP/AC recalculation.
- [ ] Expose `POST /api/characters/{id}/reroll-ability-scores` and `PATCH /api/characters/{id}/reroll-permission` in `PlayerCharacterController.java`.
- [ ] Update `dungeoneer-frontend/app/characters/[id]/edit/page.tsx` (and character sheet / campaign DM controls) to allow free rerolling when unbound (`!character.campaignId`) and DM-gated rerolling when bound to a campaign.

## Test Tasks

- [ ] Add unit tests in `CreateCharacterServiceTest` verifying non-ORIGIN feats are rejected as `originFeatId`.
- [ ] Add unit tests in `RollServiceTest` / `UpdateCharacterServiceTest` covering unbound reroll, blocked campaign reroll (`allowAbilityReroll = false`), and authorized campaign reroll (`allowAbilityReroll = true`).
- [ ] Add frontend unit tests verifying Origin Feat filter in the creation wizard and reroll button state.

## Documentation Tasks

- [ ] Update delta specs in `specs/player-character-domain-delta.md`, `specs/player-character-data-model-delta.md`, and `specs/frontend-web-app-delta.md`.

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing (`mvn clean test` and `npm test`)
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
