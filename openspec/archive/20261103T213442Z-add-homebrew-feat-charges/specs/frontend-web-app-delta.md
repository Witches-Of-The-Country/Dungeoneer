# Delta Spec: `frontend-web-app`

> **Change ID:** `add-homebrew-feat-charges`  
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`  
> **Change Type:** Modify

---

## 1. UI/UX & Client Contract Updates

### 1.1 Homebrew Feat Creation Dialog (`CreateHomebrewDialog`)
- **Charge System Section**:
  - Checkbox toggle: `Habilitar Sistema de Cargas e Descanso` (`feat-has-charges-toggle`).
  - Max charges formula selector (`feat-max-charges-type-select`):
    - `Número Fixo` (`FIXED`)
    - `Modificador de Atributo (mín. 1)` (`ABILITY_MODIFIER`)
    - `Bônus de Proficiência` (`PROFICIENCY`)
    - `Nível do Personagem` (`LEVEL`)
  - Ability selector (`feat-charge-ability-select`): active when `ABILITY_MODIFIER` is chosen (STR, DEX, CON, INT, WIS, CHA).
  - Fixed quantity input (`feat-fixed-max-charges-input`): active when `FIXED` is chosen (bounded 1..99).
  - Short Rest Recovery selector (`feat-short-rest-recovery-select`):
    - `Nenhuma (Não recupera)` (`NONE`) [default]
    - `Todas as Cargas (Total)` (`ALL`)
    - `Quantidade Fixa` (`FIXED`)
    - `Modificador do Atributo` (`MODIFIER`)
    - `Bônus de Proficiência` (`PROFICIENCY`)
    - `Nível do Personagem` (`LEVEL`)
  - Long Rest Recovery selector (`feat-long-rest-recovery-select`):
    - `Todas as Cargas (Total - Padrão)` (`ALL`) [default]
    - `Nenhuma` (`NONE`)
    - `Quantidade Fixa` (`FIXED`)
    - `Modificador do Atributo` (`MODIFIER`)
    - `Bônus de Proficiência` (`PROFICIENCY`)
    - `Nível do Personagem` (`LEVEL`)
  - Hydration on edit (`editEntity`): restores existing charge configuration and falls back to legacy `maxUses`/`rechargeOn` if present.

### 1.2 Character Sheet Feat Tab (`FeatsTab`)
- Displays live charges tracker: `{currentCharges} / {maxCharges}` with single-click consumption (`Zap` button).
- Visual badges for recovery rules:
  - Formula indicator: e.g. `Fórmula: Mod. SAB` or `Fórmula: Nível`.
  - Short Rest indicator: `Descanso Curto: Total` or `Descanso Curto: +1`.
  - Long Rest indicator: `Descanso Longo: Total` or `Descanso Longo: +1`.
- `FeatDetailModal`: displays charge formula and per-rest recovery mechanics in active mechanics panel.
