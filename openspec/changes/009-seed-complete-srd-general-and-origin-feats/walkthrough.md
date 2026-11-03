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
- **Próximos Passos (Fase 2)**: Suporte de persistência e domínio no backend:
  - Validação da desserialização completa de `mechanical_effects` em `FeatJpaEntity.java` via Jackson `ObjectMapper`.
  - Verificação e adição de testes em `MechanicalEffectsTest.java` para garantir que `spellChoice`, `prerequisiteRules` e `abilityBonusChoice` são serializados e desserializados sem perda de dados.
