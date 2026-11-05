# Tasks: Fix Legacy Items, Armors, Feat Mechanics & Rest Charge Recovery

> **Change ID:** `002-fix-legacy-items-and-feats-mechanics`
> **Execution Order:** `002`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy Data & Schema Migration Tasks

- [ ] Create `V29__fix_legacy_items_and_feats_mechanics.sql` to populate `ArmorMechanics` for `half-plate-armor-2014/2024`, `padded-armor-2014/2024`, `hide-armor-2014/2024`, `scale-mail-2014/2024`, `ring-mail-2014/2024`, and `splint-armor-2014/2024`.
- [ ] In `V29`, populate `WondrousMechanics` JSONB (`abilityOverrides`, `abilityBonuses`, `acBonus`, `savingThrowBonus`) for SRD wondrous items (Gauntlets of Ogre Power, Headband of Intellect, Amulet of Health, Belts of Giant Strength, Ioun Stones, Manuals/Tomes, Cloak/Ring of Protection, Bracers of Defense).
- [ ] In `V29`, normalize legacy `maxUses` and `rechargeOn` JSONB keys in `player_character.feats` (Channel Divinity, Wild Shape, Ki/Focus, Divine Intervention, Arcane Recovery, Font of Magic, Combat Superiority, Action Surge 2, Indomitable, etc.) and backfill `max_charges` and `current_charges` in `player_character.character_feats` for existing characters.

## Implementation Tasks

- [ ] Create `WondrousMechanics.java` and permit it in `ItemMechanics.java`; update `ItemJpaEntity.java` serialization/deserialization.
- [ ] Update `MechanicalEffects.java` and `FeatJpaEntity.java` to support legacy `maxUses` and `rechargeOn` keys via Jackson aliases/fallbacks.
- [ ] Update `CharacterSheetHydrator.java` to compute effective `AbilityScores` (applying item bonuses and `Math.max` overrides) and passive `acBonus` from equipped (and attuned where required) items.
- [ ] Fix `RestService.java` (`takeRest` and `recoverFeats`) so recovered `CharacterFeat` instances are attached to `characterToSave` via `characterToSave.withFeats(recoveredFeats)` before calling `characterRepository.save(characterToSave)`.
- [ ] Update `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` and `dice-roller-modal.tsx` so rollable feats with charges (e.g., Bardic Inspiration) spend a charge when rolled and also show the "Gastar" button.

## Test Tasks

- [ ] Add unit tests in `CharacterSheetHydratorTest` verifying Half Plate AC, Wondrous item ability overrides/bonuses, and Channel Divinity action hydration.
- [ ] Add unit/integration tests in `RestServiceTest` verifying Short and Long Rest feat charge recovery persists without cascade overwrite.
- [ ] Add frontend unit tests for `ActionsTab` and `DiceRollerModal` verifying charge consumption on rollable feat actions.

## Documentation Tasks

- [ ] Update delta specs in `specs/player-character-domain-delta.md`, `specs/player-character-data-model-delta.md`, and `specs/frontend-web-app-delta.md`.

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing (`mvn clean test` and `npm test`)
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
