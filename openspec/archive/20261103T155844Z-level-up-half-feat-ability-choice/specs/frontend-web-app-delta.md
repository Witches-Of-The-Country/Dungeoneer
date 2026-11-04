# Delta: Frontend Web App — Level-Up Wizard Half-Feat Ability Selector

> **Target Specification:** `openspec/specs/frontend-web-app.md`  
> **Change ID:** `level-up-half-feat-ability-choice`

---

## UI / Component Updates: `LevelUpWizardModal`

### Step 3: Feat Selection & Half-Feat Ability Choice
1. **Detection**:
   - When `asiOrFeatChoice === 'FEAT'` and a feat is selected, inspect `selectedFeat.mechanicalEffects?.abilityBonusChoice`.
2. **Interactive Selection Widget**:
   - If `abilityBonusChoice` is present:
     - Render a dedicated sub-panel: "Bônus de Atributo do Talento (+1)".
     - Display single-select chips/radios for each ability in `allowedAbilities` (translated using `ABILITY_LABELS` to Portuguese, e.g., `FOR`, `DES`).
     - Display current character score and projected score (+1, max 20).
   - If fixed `abilityBonuses` are present:
     - Display an informational badge indicating the automatic stat increase (e.g., `+1 Destreza`).
3. **Validation & Gating**:
   - The "Avançar" button in Step 3 blocks progression with a descriptive toast (`toast.error('Selecione o atributo para o bônus do talento (+1).')`) if a half-feat requiring an ability choice does not have a selection.
4. **Summary (Step 4)**:
   - Displays the feat name and the chosen attribute boost (e.g., `Talento: Atleta (+1 Força)`).
5. **Mutation Payload**:
   - Populates `abilityScoreIncreases: { [chosenStat]: 1 }` alongside `selectedFeatId` in the `useLevelUp` mutation payload.
