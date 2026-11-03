# Walkthrough: Add Long & Short Rest Mechanics

> **Change ID:** `add-long-short-rest`  
> **Status:** Em Progresso (Fase 1 Concluída)  
> **Data:** 2026-11-03  

---

## 📋 Resumo da Execução — Fase 1 (Migração Flyway e Fundação de Domínio Puro)

Na Fase 1, estabelecemos a infraestrutura de dados e as regras de domínio fundamentais para o rastreamento de exaustão e controle de Dados de Vida (Hit Dice) gastos e disponíveis, preparando o terreno para o motor de descanso (Fase 2).

### 1. Arquivos Criados e Modificados

| Arquivo | Camada | Modificação |
|---|---|---|
| [`V27__add_exhaustion_and_hit_dice_tracking.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V27__add_exhaustion_and_hit_dice_tracking.sql) | Migração Flyway | Adicionada coluna `exhaustion` (com restrição CHECK de 0 a 6) na tabela `characters` e coluna `hit_dice_spent` (com restrição CHECK >= 0) na tabela `class_progressions`. |
| [`PlayerCharacter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java) | Domínio | Adicionado atributo e validação de `exhaustion` (0 a 6), métodos `withExhaustion`, `addExhaustion`, `removeExhaustion`, `isDeadFromExhaustion`, `withClassProgressions`, propagação de exaustão através de todas as mutações imutáveis (`withCurrentHp`, `withLevelUp`, etc.) sem imports de framework. |
| [`ClassProgression.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ClassProgression.java) | Domínio | Adicionado atributo `hitDiceSpent`, validação de limites `0 <= hitDiceSpent <= classLevel`, métodos `spendHitDice(count)`, `recoverHitDice(count)`, `getAvailableHitDice()` e `withHitDiceSpent`. |
| [`PlayerCharacterJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java) | Persistência | Mapeamento JPA bidirecional da coluna `exhaustion` em `toDomain()` e `fromDomain()`. |
| [`ClassProgressionJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ClassProgressionJpaEntity.java) | Persistência | Mapeamento JPA bidirecional da coluna `hit_dice_spent` em `toDomain()` e `fromDomain()`. |
| [`ClassProgressionTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/ClassProgressionTest.java) | Testes Unitários | 7 novos testes unitários validando ciclo de vida, gastos, recuperação e rejeição de limites de Dados de Vida. |
| [`PlayerCharacterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacterTest.java) | Testes Unitários | Testes unitários para valores de exaustão, limites (0-6), morte por exaustão no nível 6 e preservação em mutações. |
| [`tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/add-long-short-rest/tasks.md) | Documentação | Atualizadas tarefas da Fase 1 como concluídas `[x]`. |

---

## 🧪 Verificação e Resultados dos Testes — Fase 1

```bash
# Execução dos testes unitários de domínio:
mvn test -f player-character-service/backend/pom.xml "-Dtest=PlayerCharacterTest,ClassProgressionTest"
# Resultado: Tests run: 32, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS
```

---

## 🎯 Contexto para a Fase 2 (Estratégia de Descanso no RulesEngine e RestService)

Na **Fase 2**:
1. Adicionar métodos de descanso no contrato de estratégia `RulesEngine` (`resolveShortRest`, `resolveLongRest`) com implementações especializadas para `SRD_2014` e `SRD_2024`.
2. Reescrever o `RestService` para orquestrar:
   - Descanso Curto: gasto de Dados de Vida, rolagem de cura + mod CON (mínimo 0), recuperação de cargas de talentos com regra de descanso curto.
   - Descanso Longo: cura total de PV, recuperação de metade dos Dados de Vida totais (mínimo 1), redução de 1 nível de exaustão, expiração de PV temporários, barreira contra ressurreição a 0 PV e recuperação de cargas de talentos (padrão `ALL`).
