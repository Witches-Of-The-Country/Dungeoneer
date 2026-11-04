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

---

## Fase 3: Validação na Criação de Personagem e no Level-Up

### 1. Arquivos Envolvidos
- `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterServiceTest.java`
  - Adicionado caso de teste unitário `shouldRejectOriginFeatWhenPrerequisitesNotMet` validando que `CreateCharacterService` bloqueia a seleção de talentos de origem (ou gerais) caso as regras dinâmicas de pré-requisito (`prerequisiteRules`, como valor mínimo de atributo) não sejam satisfeitas.
- `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataControllerTest.java`
  - Adicionado caso de teste de controlador Web MVC `shouldFilterFeatsByFeatTypesAndReturnMechanicalEffects` assegurando que `/api/reference/feats?featTypes=ORIGIN&featTypes=GENERAL` retorna a listagem filtrada de talentos com suas estruturas completas de `mechanicalEffects` (incluindo `prerequisiteRules`).

### 2. Resultados de Verificação e Testes
- **Comando Executado**: `mvn -f player-character-service/backend/pom.xml test`
- **Resultado**: `BUILD SUCCESS`
- **Sumário de Testes**: 231 testes executados (2 novos testes cobrindo validação de pré-requisito e endpoint de talentos), 0 falhas, 0 erros, 1 ignorado.

---

## Fase 4 & 5: Verificação do Frontend, Suíte Completa e Deploy em Contêiner

### 1. Testes Unitários do Frontend
- **Comando Executado**: `npm --prefix dungeoneer-frontend run test:unit`
- **Resultado**:
  - `Test Files: 19 passed (19)`
  - `Tests: 70 passed (70)`
  - Destaque: `level-up-wizard.test.tsx` e `character-wizard.test.tsx` validaram o seletor de talentos gerais e de origem, o filtro de categorias e a integração de dados de referência.

### 2. Build de Produção do Frontend
- **Comando Executado**: `npm --prefix dungeoneer-frontend run build`
- **Resultado**: `Compiled successfully` com todas as 17 rotas estáticas e dinâmicas geradas sem erros de tipagem TypeScript.

### 3. Rebuild e Execução do Contêiner Docker
- **Comando Executado**: `docker compose build player-character-service; docker compose up -d player-character-service`
- **Resultado**:
  - Imagem `dungeoneer-player-character-service` compilada e empacotada com sucesso.
  - Migração Flyway `V26__seed_complete_srd_general_and_origin_feats.sql` aplicada com sucesso no PostgreSQL compartilhado.
  - Endpoints REST validados em tempo real no contêiner:
    - `/api/reference/feats?featTypes=ORIGIN`: 10 talentos de origem retornados (Alerta, Artesão, Atacante Selvagem, Brigão de Taverna, Curandeiro, Habilidoso, Iniciado em Magia, Músico, Sortudo, Vigoroso) com metadados `spellChoice` e bônus de ação estruturados.
    - `/api/reference/feats?featTypes=GENERAL&limit=100`: 88 talentos gerais retornados com descrições, regras de pré-requisito e bônus.
    - `/api/reference/progression-feats`: Níveis 1 a 20 de progressão de classe (incluindo Monge Níveis 5–20) validados e retornando características completas.


