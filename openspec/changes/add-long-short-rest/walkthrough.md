# Walkthrough: Add Long & Short Rest Mechanics

> **Change ID:** `add-long-short-rest`  
> **Status:** Em Progresso (Fases 1 e 2 Concluídas)  
> **Data:** 2026-11-03  

---

## 📋 Resumo da Execução — Fase 1 (Migração Flyway e Fundação de Domínio Puro)

Na Fase 1, estabelecemos a infraestrutura de dados e as regras de domínio fundamentais para o rastreamento de exaustão e controle de Dados de Vida (Hit Dice) gastos e disponíveis, preparando o terreno para o motor de descanso (Fase 2).

### 1. Arquivos Criados e Modificados na Fase 1

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

## 📋 Resumo da Execução — Fase 2 (RulesEngine Rest Mechanics & RestService)

Na Fase 2, implementamos as regras e cálculos de descanso delegados via Strategy Pattern no `RulesEngine`, além da reescrita completa do `RestService`.

### 1. Arquivos Criados e Modificados na Fase 2

| Arquivo | Camada | Modificação |
|---|---|---|
| [`RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RulesEngine.java) | Domínio | Métodos adicionados: `calculateHitDieHealing(dieRoll, constitutionModifier)`, `calculateHitDiceRecovery(totalCharacterLevel)`, `calculateExhaustionRecovery(currentExhaustion)`, e factory helper estático `RulesEngine.forRuleset(ruleset)`. |
| [`Srd2014RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2014RulesEngine.java) | Domínio | Implementação pura de regras de descanso SRD 2014: cura de dado de vida com mod CON (mínimo 0 por dado), recuperação de metade do nível total de dados de vida arredondado para baixo (mínimo 1 dado), redução de exaustão em 1 (mínimo 0). |
| [`Srd2024RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2024RulesEngine.java) | Domínio | Implementação para SRD 2024 delegando coerentemente ao motor base. |
| [`RestCommand.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/RestCommand.java) | Aplicação (DTO) | Adicionado record interno `HitDiceSpendRequest(classProgressionId, count)` com validações Jakarta (`@NotNull`, `@Min(1)`), mantendo retrocompatibilidade total com construtores anteriores. |
| [`RestService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RestService.java) | Aplicação (Serviço) | Reescrita completa do orquestrador de descanso: barreira de segurança contra ressurreição (personagem a 0 HP rejeitado com 422), preservação estrita de HP temporário no Descanso Longo, rolagem autoritativa de dados pelo servidor via `DiceRoller`, aplicação e capeamento de cura no HP máximo, controle atômico de dados de vida gastos/recuperados por progressão de classe, redução de exaustão, e recarga de cargas de talentos via `FeatChargeCalculator`. |
| [`RulesEngineTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/RulesEngineTest.java) | Testes de Domínio | Adicionado caso de teste (TE-07, TE-05, TE-06) cobrindo cálculos de cura com mod CON negativo/positivo, recuperação de dados para níveis 1 a 20 e limites de exaustão. |
| [`RestServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RestServiceTest.java) | Testes de Serviço | 9 testes unitários determinísticos cobrindo TP-01, TP-02, TE-01, TE-02, TE-03, TE-04, TE-05, TE-06 e recuperação de cargas de talentos em descanso curto e longo. |
| [`tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/add-long-short-rest/tasks.md) | Documentação | Atualizadas tarefas da Fase 2 como concluídas `[x]`. |

---

## 🧪 Verificação e Resultados dos Testes — Fase 2

```bash
# Execução dos testes de descanso e motor de regras:
mvn test -f player-character-service/backend/pom.xml "-Dtest=*Rest*Test,*RulesEngineTest"
# Resultado: Tests run: 21, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS

# Execução da suíte completa de serviços e domínio:
mvn test -f player-character-service/backend/pom.xml "-Dtest=*ServiceTest,*Progression*Test,*RulesEngine*Test"
# Resultado: Tests run: 113, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS
```

---

## 🎯 Contexto para a Fase 3 (Controlador REST, DTOs & Testes de Integração)

Na **Fase 3**:
1. Atualizar portas e DTOs de resposta da ficha (`CharacterSheetResult`, `CharacterClassSummaryDto`) para expor `exhaustion`, `hitDiceSpent` e `availableHitDice`.
2. Atualizar o `PlayerCharacterController` com o endpoint `POST /api/v1/characters/{id}/rest`.
3. Validar RFC 7807 `ProblemDetail` para casos de erro (personagem morto/0 HP, gastar mais dados do que disponíveis, ID de progressão inválido).
4. Implementar testes MockMvc (`PlayerCharacterControllerTest`) e testes de integração com Testcontainers (`PlayerCharacterPersistenceIntegrationTest`).
