# Delta Spec: `frontend-web-app`

> **Change ID:** `002-fix-legacy-items-and-feats-mechanics`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### Charged Actions & Dice Roller Charge Deduction
- In `ActionsTab`, any feat action with `maxCharges > 0` renders the "Gastar" button even when `isRollable` is true.
- When a user clicks "Rolar" on a feat action that has charges (`maxCharges > 0` and `currentCharges > 0`), `DiceRollerModal` invokes `useFeatMutation` to decrement 1 charge upon rolling.
