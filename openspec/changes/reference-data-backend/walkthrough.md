# Walkthrough — Mudança `reference-data-backend`

> **Change ID:** `reference-data-backend`  
> **Status:** Em Progresso (Fase 1 Concluída)  
> **Data:** 2026-10-30

---

## 📌 Progresso Acumulado das Fases

### Fase 1: Migrações Flyway e Alinhamento de Schema JPA (✅ Concluída)
- **O que foi feito:**
  - Criada a migração [`V2__align_schema_and_create_reference_tables.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V2__align_schema_and_create_reference_tables.sql):
    - Criação do schema isolado `player_character`.
    - Movimentação das tabelas legadas `characters` e `ability_scores` do schema `public` para `player_character`.
    - Criação das tabelas de dados de referência: `player_character.archetypes`, `player_character.lineages` e `player_character.backgrounds`, incluindo colunas para `JSONB` de ASI e sumário de traços e talentos de origem do D&D 2024.
    - Adição de chaves estrangeiras `fk_characters_lineage` e `fk_characters_background`.
  - Criada a migração [`V3__seed_srd_2014_and_2024_reference_data.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V3__seed_srd_2014_and_2024_reference_data.sql):
    - Semente de todas as **13 Classes** do SRD (Bárbaro, Bardo, Clérigo, Druida, Guerreiro, Monge, Paladino, Guardião, Ladino, Feiticeiro, Bruxo, Mago, Artífice) para `SRD_2014` e `SRD_2024` (26 registros).
    - Semente de todas as **9 Linhagens** do SRD (Humano, Elfo, Anão, Halfling, Gnomo, Meio-Elfo, Meio-Orc, Tiefling, Draconato) para `SRD_2014` (com ASI em JSONB) e `SRD_2024` (com traços atualizados e ASI no antecedente) (18 registros).
    - Semente de todos os **6 Antecedentes** do SRD (Acólito, Criminoso, Herói do Povo, Nobre, Sábio, Soldado) para `SRD_2014` e `SRD_2024` (com talentos de origem e 3 opções de bônus de atributo) (12 registros).
  - Atualizadas as entidades JPA [`PlayerCharacterJpaEntity`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java) e [`AbilityScoresJpaEntity`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/AbilityScoresJpaEntity.java) para apontar para `schema = "player_character"`.
  - Atualizado o teste de integração [`PlayerCharacterPersistenceAdapterTest`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterPersistenceAdapterTest.java) para utilizar IDs semeados, respeitando as constraints de integridade referencial.
- **Validação:**
  - `mvn -f player-character-service/backend/pom.xml test` executado com sucesso: 48 testes passando, 0 falhas, 0 erros.

---

### Próxima Etapa: Fase 2 (Modelos de Domínio Puros em Java com TDD)
- Criação dos modelos de domínio `Archetype`, `Lineage`, `Background`, `HitDiceEnum`, `SpellcastingType` na camada `domain/model` sem qualquer import de framework.
- Criação da suíte de testes unitários de domínio cobrindo invariantes e métodos auxiliares.
