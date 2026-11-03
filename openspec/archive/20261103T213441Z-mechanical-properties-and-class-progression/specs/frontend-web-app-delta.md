# Delta Spec: `frontend-web-app`

> **Change ID:** `mechanical-properties-and-class-progression`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Character Creation Wizard (`/characters/new`) & Character Sheet Header (`/characters/[id]`)
- **Character Creation Wizard (`app/characters/new/page.tsx`)**:
  - **Primary Class Only**: Allows selecting target starting level (`1`–`20`) and **only the primary class** (multiclass configuration is removed from creation and delegated to the Level Up Wizard).
  - **Career HP Method (`hpCalculationMethod`)**: Always selectable (`AVERAGE`, `ROLLED`, or `MAX`) in Step 3 (`Classe`) regardless of whether starting at Level 1 or higher (Level 1 always grants maximum Hit Die + CON modifier, while the chosen method is persisted and locked for Levels 2–20).
  - **Level 1 Subclass (`subclassLevel == 1`)**: Renders the Subclass selector in Step 3 (`Classe`) **only** when the selected primary class has `subclassLevel === 1` (e.g., `SRD_2014` Cleric, Sorcerer, and Warlock, or a Homebrew class with `subclassLevel == 1`).
  - **Level 1 Origin Feat (`SRD_2024`)**: When `ruleset === "SRD_2024"`, Step 4 (`Antecedente`) allows choosing 1 `ORIGIN` feat (`SRD_2024` + `HOMEBREW` of type `ORIGIN`).
- **`XpProgressBar` (`components/character-sheet/xp-progress-bar.tsx`) & Pending Level-Up Auto-Prompt**:
  - Renders current XP vs next level threshold (e.g., `1.200 / 2.700 XP — Nível 3 → 4`) using an Arcane Gold (`#C8AA6E`) progress bar.
  - Displays a `+ Adicionar XP` button that opens `AddXpModal` (disabled with tooltip `"XP controlado pelo Mestre da Campanha"` when `character.campaignId != null`).
  - Displays a pulsing `"Subir de Nível!"` CTA button when `experiencePoints >= nextLevelXp` or `character.hasPendingLevelChoices === true`, plus a `"Evoluir por Marco"` option inside `AddXpModal`.
  - When landing on `/characters/[id]`, if `experiencePoints >= nextLevelXp` or `hasPendingLevelChoices === true`, `LevelUpWizardModal` opens automatically in sequence (`Nv. 1 -> 2 -> ... -> N`), and the `"Escolhas de Nível Pendentes"` badge on the sheet header is also clickable to reopen the modal.

### 2. Multi-Step Level Up Wizard (`components/character-sheet/level-up-wizard-modal.tsx`)
- **Step 1 — Classe & Multiclasse**:
  - Interactive class selector allowing the player to advance an existing class (`Guerreiro Nv. 2 → 3`) or add any new class from the character's ruleset + Homebrew (`Multiclasse Livre`, with zero ability score prerequisite locks).
- **Step 2 — Subclasse (Conditional)**:
  - Shown when the selected target class reaches its `subclassLevel` (`newClassLevel === targetArchetype.subclassLevel`). Lists available SRD and Homebrew Subclasses for that archetype with descriptions and level features.
- **Step 3 — Pontos de Vida (PV) — Travado no Método da Criação**:
  - Enforces the character's persisted `character.hpCalculationMethod` (no method switcher in the modal):
    - If `AVERAGE` or `MAX`, displays the deterministic HP breakdown (`Dado da Classe Alvo [Média/Máx] + Mod. CON + Bônus de Talentos`).
    - If `ROLLED`, displays the server-validated Hit Die roller button (`POST /api/characters/rolls/hp`) + `Mod. CON + Bônus de Talentos`.
- **Step 4 — Características Desbloqueadas & ASI / Talento**:
  - Lists all `CLASS` and `SUBCLASS` features automatically unlocked at the new class level.
  - If the new class level is an ASI threshold (`4, 8, 12, 16, 19`, Fighter `6, 14`, Rogue `10`), renders a toggle between **Aumento no Valor de Atributo (+2 ou +1/+1, máx. 20)** and **Escolher um Talento** (`GENERAL` and `ORIGIN` feats from character's ruleset + Homebrew, excluding already-owned feats, displaying `prerequisites` as an informational badge without hard-blocking).
  - Sanitizes optional `UUID` fields (`subclassId`, `hpRollId`, `selectedFeatId`) as `null`/`undefined` (never `""`).

### 3. Activated Character Sheet Tabs (`/characters/[id]`)
- **Aba `Inventário` (`components/character-sheet/inventory-tab.tsx`)**:
  - **Header**: Total weight carried (`Peso Total`) and Attunement counter (`Sintonização: X / 3`).
  - **Add Item Modal/Drawer**: Searchable catalog of SRD + Homebrew items filtered by ruleset and category (`Arma`, `Armadura`, `Consumível`, `Mágico`, `Equipamento`).
  - **Item Cards/Rows**:
    - Displays mechanical badges (e.g., `1d8 Cortante | Versátil (1d10)`, `CA 14 + DES (máx 2) | Armadura Média`, `CA 12 + DES | Veste (Robe)`, `+2 CA | Escudo`, `Cura 2d4+2 PV`).
    - **Equipar / Desequipar** toggle for weapons, armor, robes, and shields (triggers immediate AC and Actions recalculation).
    - **Sintonizar** toggle for items with `attunement = true` (blocked at 3/3).
    - **Usar** button for `CONSUMABLE` items that triggers `POST /api/characters/{id}/items/{characterItemId}/use`, displays a toast with the dice roll and HP healed/granted, and decrements quantity.
- **Aba `Talentos` (`components/character-sheet/feats-tab.tsx`)**:
  - Top action bar with **Descanso Curto** and **Descanso Longo** buttons.
  - Grouped sections: **Talentos de Origem & Gerais**, **Características de Classe**, **Características de Subclasse**, and **Traços de Linhagem**.
  - Features with `usesPerRest` display interactive charge slots (`● ● ○ — 2/3 usos | Descanso Curto`) and an `"Usar Habilidade"` button.
- **Aba `Ações` (`components/character-sheet/actions-tab.tsx` & `dice-roller-modal.tsx`)**:
  - **Ataques com Armas**: Auto-generated rows for every equipped weapon showing Attack Bonus (`+7 para acertar`), Primary/Secondary Damage (`1d8+4 Cortante + 1d6 Fogo`), and quick roll buttons (`Rolar Ataque`, `Rolar Dano`, `Rolar Versátil 2 Mãos`).
  - **Ações de Classe & Talentos**: Lists active abilities from unlocked feats/class features (`Second Wind`, `Rage`, `Sneak Attack`, etc.) with one-click roll and charge consumption.

### 4. Dynamic Homebrew Creation (`components/homebrew/create-homebrew-dialog.tsx`) & Class Encyclopedia (`/encyclopedia/classes`)
- **`CreateHomebrewDialog`**:
  - Adds **Subclasse** as a 6th creation tab (linked to a parent `Archetype` with level-gated subclass features).
  - When creating an **Item**:
    - `WEAPON`: Dynamic list of damage rolls (`diceFormula` + `damageType`), optional `versatileDice`, `magicBonus`, and multi-select `WeaponProperty` chips.
    - `ARMOR`: Select `armorCategory` (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), `baseAc`, `magicBonus`, `strengthRequirement`, and `stealthDisadvantage`.
    - `CONSUMABLE`: Select `effectType` (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`), `diceFormula`, and effect description.
  - When creating a **Talento (`Feat`)**: Configure `featType` (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`), passive bonuses (`acBonus`, `initiativeBonus`, `speedBonus`, `hpPerLevelBonus`), ability score bonuses, or active action (`actionName`, `actionType`, `diceFormula`, `usesPerRest`, `restType`).
  - When creating a **Classe (`Archetype`)** or **Subclasse**: Configure `subclassLevel` (on Classe), parent `archetypeId` (on Subclasse), and a **hybrid Level 1–20 feature builder** that allows either creating new level-gated features inline (with `unlockLevel`, `name`, `description`, and optional `mechanicalEffects`) or linking existing catalog features.
- **`/encyclopedia/classes` & `ClassDetailModal` (`components/encyclopedia/class-detail-modal.tsx`)**:
  - Clicking any class card in `/encyclopedia/classes` opens `ClassDetailModal` displaying saving throws, armor/weapon proficiencies, `subclassLevel`, the **Tabela de Progressão (Níveis 1 a 20)** (`Nível`, `Bônus de Proficiência`, expandable `Características de Classe` with descriptions and mechanical badges), and an expandable **Subclasses** section with level-by-level subclass features.
- **Toggleable `DUNGEON_MASTER` Active Role (`/profile` & `Navbar`)**:
  - In `app/profile/page.tsx` ("Papéis Ativos") and `components/navbar.tsx` (profile dropdown menu), `Jogador (PLAYER)` is shown as the fixed base role while `Mestre de Jogo (DUNGEON_MASTER)` renders an interactive toggle switch calling `POST /api/auth/me/roles/toggle-dm` via `useAuth().toggleDM()`, immediately updating the stored JWT and UI permissions.
