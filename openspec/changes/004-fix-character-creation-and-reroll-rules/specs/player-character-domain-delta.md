# Delta Spec: `player-character-domain`

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Origin Feat Enforcement on Character Creation
- `CreateCharacterService` enforces `feat.getFeatType() == FeatType.ORIGIN` whenever `originFeatId` is provided during Level 1 character creation.

### Ability Score Reroll Governance
- Unbound characters (`campaignId == null`) can reroll ability scores at any time.
- Campaign-bound characters (`campaignId != null`) can only reroll ability scores when `allowAbilityReroll == true` (set by the Campaign DM). Once a campaign-bound character completes a reroll, `allowAbilityReroll` resets to `false`.
