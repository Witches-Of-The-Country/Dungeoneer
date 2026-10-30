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

### Fase 2: Modelos de Domínio Puros em Java com TDD (✅ Concluída)
- **O que foi feito:**
  - Criados os enums de domínio:
    - [`HitDiceEnum.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/HitDiceEnum.java): enum com valores de dados de vida (`D6`, `D8`, `D10`, `D12`), cálculo de média de HP e método utilitário `fromString`.
    - [`SpellcastingType.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/SpellcastingType.java): enum com tipos de conjuração (`NONE`, `HALF`, `FULL`, `PACT`).
  - Implementadas as entidades de domínio ricas e imutáveis com **zero dependências de framework**:
    - [`Archetype.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Archetype.java): id, nome, hitDice, proficiências em salvaguardas/armaduras/armas, tipo e atributo de conjuração, ruleset, source e ownerId.
    - [`Lineage.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Lineage.java): id, nome, descrição, velocidade, tamanho, mapa de bônus de atributos (ASI), sumário de traços, ruleset, source e ownerId.
    - [`Background.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Background.java): id, nome, descrição, proficiências em perícias/ferramentas, idiomas, lista de atributos permitidos para ASI (2024), nome e descrição do talento de origem (2024), ruleset, source e ownerId.
  - Desenvolvida a suíte de testes unitários de domínio (TDD Red ➔ Green):
    - [`ArchetypeTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/ArchetypeTest.java) (5 testes)
    - [`LineageTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/LineageTest.java) (5 testes)
    - [`BackgroundTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/BackgroundTest.java) (5 testes)
- **Validação:**
  - 15 novos testes unitários executados com sucesso.
  - Suíte completa do serviço: **63 testes executados com 100% de sucesso**.

---

### Fase 3: Camada de Aplicação & Portas Hexagonais (✅ Concluída)
- **O que foi feito:**
  - Criada a porta de entrada [`GetReferenceDataUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/GetReferenceDataUseCase.java):
    - `List<ArchetypeDto> getClasses(Ruleset ruleset);`
    - `List<LineageDto> getLineages(Ruleset ruleset);`
    - `List<BackgroundDto> getBackgrounds(Ruleset ruleset);`
  - Criada a porta de saída [`ReferenceDataRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java):
    - `findArchetypesByRuleset`, `findLineagesByRuleset`, `findBackgroundsByRuleset`.
  - Criados os DTOs imutáveis records em `application/dto/`:
    - [`ArchetypeDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/ArchetypeDto.java)
    - [`LineageDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/LineageDto.java)
    - [`BackgroundDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/BackgroundDto.java)
  - Implementado o serviço de aplicação [`ReferenceDataService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/ReferenceDataService.java) (`@Service`, `@Transactional(readOnly = true)`).
  - Desenvolvida a suíte de testes unitários com Mockito [`ReferenceDataServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/ReferenceDataServiceTest.java).
- **Validação:**
  - `ReferenceDataServiceTest` executado com 100% de sucesso (3 testes passando).

---

### Fase 4: Adaptadores de Persistência JPA com Testcontainers (✅ Concluída)
- **O que foi feito:**
  - Implementadas as entidades JPA com mapeamento explícito para `schema = "player_character"`:
    - [`ArchetypeJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ArchetypeJpaEntity.java): mapeia colunas de proficiências (`text[]`), `hit_dice`, `spellcasting_type`, com métodos de conversão bidirecional `toDomain()` e `fromDomain()`.
    - [`LineageJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/LineageJpaEntity.java): mapeia `ability_bonuses` (`jsonb`) com `@JdbcTypeCode(SqlTypes.JSON)` e `traits_summary` (`text[]`), com `toDomain()` e `fromDomain()`.
    - [`BackgroundJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/BackgroundJpaEntity.java): mapeia proficiências e campos D&D 2024 (`allowed_ability_bonuses`, `origin_feat_name`, `origin_feat_description`), com `toDomain()` e `fromDomain()`.
  - Criadas as interfaces Spring Data JPA:
    - [`SpringDataArchetypeRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataArchetypeRepository.java)
    - [`SpringDataLineageRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataLineageRepository.java)
    - [`SpringDataBackgroundRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataBackgroundRepository.java)
  - Implementado o adaptador hexagonal [`ReferenceDataPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java):
    - Implementa `ReferenceDataRepository`.
    - Trata filtro de `ruleset` nulo consultando `findAll()`, ou filtra por `findByRuleset(ruleset)`.
  - Desenvolvida a suíte de testes de integração com Testcontainers:
    - [`ReferenceDataPersistenceAdapterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapterTest.java): valida TP-02 garantindo as contagens exatas das sementes das migrações `V2` e `V3` (13 classes, 9 linhagens, 6 antecedentes por ruleset; 26, 18 e 12 no total).
- **Validação:**
  - `mvn -f player-character-service/backend/pom.xml test` executado com sucesso: **69 testes passando, 0 falhas, 0 erros**.

---

### Próxima Etapa: Fase 5 (Adaptador Web REST & Endpoints da API)
- Implementação de `ReferenceDataController` em `adapter/in/web/`:
  - `GET /api/reference/classes`
  - `GET /api/reference/lineages`
  - `GET /api/reference/backgrounds`
  - Suporte ao query parameter opcional `?ruleset=SRD_2014|SRD_2024`.
  - Validação estrita de `ruleset` com mapeamento para `422 Unprocessable Entity` RFC 7807 `ProblemDetail`.
- Testes MockMvc (`ReferenceDataControllerTest`) cobrindo TP-03 e TE-01.


