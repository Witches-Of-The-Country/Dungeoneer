# Walkthrough: 009-seed-complete-srd-general-and-origin-feats

## Fase 1: Migração de Banco de Dados V26 (Talentos Gerais e de Origem do SRD)

### 1. Arquivos Envolvidos
- `player-character-service/backend/src/main/resources/db/migration/V26__seed_complete_srd_general_and_origin_feats.sql`
  - Inserção de 40+ talentos gerais do SRD 2014 com faixas de UUID determinísticas (`30000000-0000-0000-0000-0000000007xx`).
  - Inserção de 30+ talentos gerais do SRD 2024 com faixas de UUID determinísticas (`40000000-0000-0000-0000-0000000007xx`).
  - Inserção de talentos de origem do SRD 2024 (*Artesão*, *Curandeiro*, *Sortudo*, *Músico*, *Brigão de Taverna*) na faixa `40000000-0000-0000-0000-0000000008xx`.
  - Normalização das entradas legadas de V14 (*Iniciado em Magia*, *Atacante Selvagem*, *Habilidoso*) para `feat_type = 'ORIGIN'`.
  - Estruturação de `mechanical_effects` com regras de pré-requisitos (`prerequisiteRules`), escolhas de bônus de atributo (`abilityBonusChoice`) e metadados `spellChoice` para *Iniciado em Magia* e *Conjurador de Rituais*.
  - Idempotência assegurada via `ON CONFLICT (ruleset, slug) DO UPDATE`.

### 2. Resultados de Verificação e Testes
- **Comando Executado**: `mvn -f player-character-service/backend/pom.xml test`
- **Resultado**: `BUILD SUCCESS` (01:00 min)
- **Sumário de Testes**: 228 testes executados, 0 falhas, 0 erros, 1 ignorado.
- Todas as migrações Flyway (até a `V26`) aplicadas com sucesso pelo PostgreSQL via Testcontainers sem conflitos de chave ou integridade referencial.

### 3. Contexto de Transição para a Fase 2
- A Fase 1 está 100% concluída e validada.

---

## Fase 2: Suporte de Persistência & Domínio no Backend

### 1. Arquivos Envolvidos
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffects.java`
  - Modelo de domínio Clean Architecture (zero dependências de framework) suportando `SpellChoice`, `AbilityBonusChoice`, `FeatPrerequisite`, `conditional` e `conditionDescription`.
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/SpellChoice.java`
  - Record de domínio para seleção de magias e truques concedidos por talentos como *Iniciado em Magia* e *Conjurador de Rituais*.
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntity.java`
  - Serialização e desserialização completa de `mechanical_effects` via Jackson `ObjectMapper`, eliminando truncamentos e preservando todos os atributos.
- `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/FeatJpaEntityTest.java`
  - Testes unitários para desserialização de JSON rico e conversão round-trip completa (`serializeEffects` -> `deserializeEffects`), garantindo integridade de `spellChoice`, `abilityBonusChoice` e `prerequisiteRules`.

### 2. Resultados de Verificação e Testes
- **Comando Executado**: `mvn -f player-character-service/backend/pom.xml test`
- **Resultado**: `BUILD SUCCESS` (01:00 min)
- **Sumário de Testes**: 229 testes executados (1 novo teste de conversão round-trip), 0 falhas, 0 erros, 1 ignorado.

### 3. Contexto de Transição para a Fase 3
- A Fase 2 está 100% concluída e testada.
- **Próximos Passos (Fase 3)**: Validação na criação de personagem e no Level-Up:
  - Verificação da validação dinâmica de pré-requisitos em `CreateCharacterService` e `LevelUpService`.
  - Verificação de endpoints de talentos `/api/reference/feats` para `featTypes=GENERAL` e `featTypes=ORIGIN`.
