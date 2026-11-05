# Delta Spec: `frontend-web-app`

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Summary of Updates:** Wizard Origin Feat restriction & Ability Reroll UI

## Architectural Updates

### Creation Wizard Origin Feat Filter & Ability Reroll Controls
- In `app/characters/new/page.tsx`, the Background Origin Feat dropdown strictly filters `feat.featType === 'ORIGIN'`.
- In `app/characters/[id]/edit/page.tsx`, the "Re-rolar Atributos" workflow is active when `!character.campaignId || character.allowAbilityReroll`, and displays a DM-locked badge when `character.campaignId && !character.allowAbilityReroll`.
