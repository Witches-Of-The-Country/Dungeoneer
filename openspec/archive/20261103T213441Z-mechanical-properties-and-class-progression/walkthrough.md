# Walkthrough: Mechanical Properties, Class Progression & Character Sheet Evolution

Este documento registra o progresso acumulativo e a validação das fases de implementação da proposta `mechanical-properties-and-class-progression`.

---

## Fase 1: Migrações de Banco de Dados e Carga de Dados SRD (2014 & 2024)

### 1. Resumo das Alterações Realizadas
- **`V16__add_mechanical_properties_and_class_progression_tables.sql`**:
  - Adicionou a coluna `mechanical_properties JSONB` e índice GIN em `player_character.items`.
  - Adicionou o discriminador `feat_type VARCHAR(30)` (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`) e a coluna `mechanical_effects JSONB` em `player_character.feats`.
  - Adicionou `subclass_level INT` em `player_character.archetypes`.
  - Criou a tabela `player_character.subclasses` para gerenciar subclasses SRD e Homebrew, conectada à chave estrangeira `class_progressions.subclass_id`.
  - Criou as tabelas associativas de progressão de níveis 1 a 20: `player_character.archetype_feats` e `player_character.subclass_feats`.
  - Adicionou as colunas `experience_points INT NOT NULL DEFAULT 0` e `campaign_id UUID NULL` em `player_character.characters`.
  - Criou as tabelas de estado da ficha: `player_character.character_items` (inventário com suporte a equipar e sintonizar) e `player_character.character_feats` (talentos desbloqueados com controle de cargas e escolhas).
- **`V17__seed_srd_subclasses_and_progression_feats.sql`**:
  - Atualizou o `subclass_level` dos arquétipos (nível 3 padronizado para `SRD_2024`; níveis 1, 2 ou 3 para `SRD_2014`).
  - Cadastrou todas as 13 Subclasses SRD para `SRD_2014` e `SRD_2024`:
    - Bárbaro: *Caminho do Furioso*
    - Bardo: *Colégio do Conhecimento*
    - Clérigo: *Domínio da Vida*
    - Druida: *Círculo da Terra*
    - Guerreiro: *Campeão*
    - Monge: *Caminho da Mão Aberta*
    - Paladino: *Juramento de Devoção*
    - Patrulheiro: *Caçador*
    - Ladino: *Ladrão*
    - Feiticeiro: *Linhagem Dracônica*
    - Bruxo: *O Corruptor*
    - Mago: *Escola de Evocação*
    - Artífice: *Alquimista*
  - Cadastrou talentos de classe e de subclasse (*Retomar o Fôlego*, *Surto de Ação*, *Ataque Extra*, *Fúria*, *Ataque Furtivo*, *Destruição Divina*, *Inspiração Bárdica*, *Crítico Aprimorado*, etc.) com efeitos mecânicos e vinculou-os em `archetype_feats` e `subclass_feats`.
- **`V18__populate_srd_mechanical_properties.sql`**:
  - Povoou `mechanical_properties` em armas SRD (dados de dano, versátil, propriedades), armaduras (categorias `LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), e consumíveis (`Poção de Cura`).
  - Povoou `feat_type` e `mechanical_effects` nos talentos SRD existentes (*Vigoroso*, *Alerta*, *Defensivo*, etc.).

### 2. Verificação e Testes
- Compilação dos recursos e validação do backend via Maven:
  ```bash
  mvn -f player-character-service/backend/pom.xml test-compile
  # Resultado: BUILD SUCCESS (24 recursos processados)
  ```

---

## Fase 2: Clean Architecture Domain Value Objects, Entities & Rules Engines

### 1. Resumo das Alterações Realizadas
- **Modelos e Value Objects de Domínio Puro (Java 21, Zero Framework Imports)**:
  - `DamageType`: Enum completo com os 13 tipos de dano da 5e (`SLASHING`, `PIERCING`, `FIRE`, `RADIANT`, etc.).
  - `DamageRoll`: Record que associa fórmula de dado validada por regex e tipo de dano.
  - `WeaponProperty`: Enum com as 9 propriedades oficiais de armas (`FINESSE`, `LIGHT`, `VERSATILE`, etc.).
  - `ArmorCategory`: Enum (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`).
  - `ConsumableEffectType`: Enum (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`).
  - `ItemMechanics`: Sealed interface com implementações `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics` e `EmptyItemMechanics`.
  - `FeatType` e `RestType`: Enums para categorização de talentos e recarga de recursos.
  - `MechanicalEffects`: Value Object puro para modelagem de bônus passivos (CA, Iniciativa, PV por nível), proficiências concedidas, ações com rolagem e controle de cargas por descanso curto/longo.
  - `CharacterItem`: Modelo de item na ficha com suporte a quantidade, equipar e sintonizar (com invariante de no máximo 3 itens sintonizados).
  - `CharacterFeat`: Modelo de talento na ficha com controle de cargas e recarga seletiva em descansos curto ou longo.
  - `Subclass`: Entidade de domínio completa implementando subclasses vinculadas a arquétipos com `ruleset`, `slug`, `nameEn`, `name` e `description`.
  - `Archetype`: Atualizado com `subclassLevel` e verificação de níveis de ASI (`isAsiLevel`).
  - `Item` e `Feat`: Atualizados com propriedades mecânicas e construtores retrocompatíveis.
  - `ClassProgression`: Métodos funcionais para evolução de nível e amarração de subclasse.
  - `RulesEngine` (`Srd2014RulesEngine` e `Srd2024RulesEngine`):
    - Cálculo de Classe de Armadura cobrindo `HEAVY`, `MEDIUM`, `LIGHT`, `ROBE` (com compatibilidade e soma a Defesa sem Armadura de Bárbaro/Monge) e `SHIELD`.
    - Tabela oficial de progressão de XP 1 a 20 (0 a 355.000 XP) com métodos `getXpForLevel` e `getLevelForXp`.
  - `PlayerCharacter`: Adicionadas as colunas e métodos de domínio `addExperience`, `withLevelUp` (multiclasse livre sem trava de atributo >= 13), `withArmorClass`, `withItems` (com trava de 3 sintonizações), `withFeats`, `withCurrentHp` e `withTempHp`.

### 2. Verificação e Testes
- Execução de testes unitários do domínio:
  ```bash
  mvn -f player-character-service/backend/pom.xml test "-Dtest=ItemMechanicsTest,MechanicalEffectsTest,RulesEngineAcAndProgressionTest,RulesEngineTest,PlayerCharacterTest"
  # Resultado: Tests run: 40, Failures: 0, Errors: 0, Skipped: 0 (BUILD SUCCESS)
  ```

---

## Próxima Fase: Fase 3
- **Foco:** Persistence Adapters, Application Services & REST Controllers (JPA, JSONB via Hibernate, `LevelUpService`, `InventoryService`, `RestService` e endpoints REST).
