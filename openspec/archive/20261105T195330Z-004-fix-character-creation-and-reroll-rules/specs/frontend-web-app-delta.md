# Delta Spec: `frontend-web-app`

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Summary of Updates:** Wizard Origin Feat restriction, Wizard level input decoupling & Ability Reroll UI

## Architectural Updates

### Creation Wizard Origin Feat Filter, Level Input Decoupling & Ability Reroll Controls
- In `app/characters/new/page.tsx`, the Background Origin Feat dropdown strictly filters `feat.featType === 'ORIGIN'`.
- In `app/characters/new/page.tsx` (Step 3: Classe), the character level input decouples immediate numeric clamping from `onChange`, permitting clearing or entering multi-digit numbers (e.g. 20) during editing, and applies clamping `[1, 20]` (defaulting to 1 if blank) on `onBlur`.
- In `app/characters/[id]/edit/page.tsx`, the "Re-rolar Atributos" workflow is active when `!character.campaignId || character.allowAbilityReroll`, and displays a DM-locked badge when `character.campaignId && !character.allowAbilityReroll`.
