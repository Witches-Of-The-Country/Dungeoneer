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

## Próxima Fase: Fase 2
- **Foco:** Clean Architecture Domain Value Objects, Entities & Rules Engines (Java 21 puro, zero anotações de framework em `domain/model/`).
