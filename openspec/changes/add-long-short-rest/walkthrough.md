# Walkthrough: Add Long & Short Rest Mechanics

> **Change ID:** `add-long-short-rest`  
> **Status:** Em Progresso (Fases 1, 2 e 3 Concluídas)  
> **Data:** 2026-11-03  

---

## 📋 Resumo da Execução — Fase 1 (Migração Flyway e Fundação de Domínio Puro)

Na Fase 1, estabelecemos a infraestrutura de dados e as regras de domínio fundamentais para o rastreamento de exaustão e controle de Dados de Vida (Hit Dice) gastos e disponíveis, preparando o terreno para o motor de descanso (Fase 2).

### Arquivos Criados e Modificados na Fase 1

| Arquivo | Camada | Modificação |
|---|---|---|
| [`V27__add_exhaustion_and_hit_dice_tracking.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V27__add_exhaustion_and_hit_dice_tracking.sql) | Migração Flyway | Adicionada coluna `exhaustion` (CHECK 0-6) na tabela `characters` e coluna `hit_dice_spent` (CHECK >= 0) na tabela `class_progressions`. |
| [`PlayerCharacter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java) | Domínio | Atributo e métodos de `exhaustion`, propagação imutável sem frameworks. |
| [`ClassProgression.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ClassProgression.java) | Domínio | Métodos `spendHitDice`, `recoverHitDice`, `getAvailableHitDice`, `withHitDiceSpent`. |
| [`PlayerCharacterJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java) | Persistência | Mapeamento JPA bidirecional da coluna `exhaustion`. |
| [`ClassProgressionJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ClassProgressionJpaEntity.java) | Persistência | Mapeamento JPA bidirecional da coluna `hit_dice_spent`. |
| [`ClassProgressionTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/ClassProgressionTest.java) | Testes Unitários | 7 testes unitários de ciclo de vida de Dados de Vida. |
| [`PlayerCharacterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacterTest.java) | Testes Unitários | Testes unitários para valores e limites de exaustão. |

---

## 📋 Resumo da Execução — Fase 2 (RulesEngine Rest Mechanics & RestService)

Na Fase 2, implementamos as regras e cálculos de descanso delegados via Strategy Pattern no `RulesEngine`, além da reescrita completa do `RestService`.

### Arquivos Criados e Modificados na Fase 2

| Arquivo | Camada | Modificação |
|---|---|---|
| [`RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RulesEngine.java) | Domínio | Métodos de cálculo de descanso: `calculateHitDieHealing`, `calculateHitDiceRecovery`, `calculateExhaustionRecovery`, e factory helper `RulesEngine.forRuleset`. |
| [`Srd2014RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2014RulesEngine.java) | Domínio | Regras SRD 2014 para descanso curto e longo. |
| [`Srd2024RulesEngine.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2024RulesEngine.java) | Domínio | Regras SRD 2024 para descanso curto e longo. |
| [`RestCommand.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/RestCommand.java) | Aplicação (DTO) | Record `HitDiceSpendRequest(classProgressionId, count)` com validações Jakarta. |
| [`RestService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RestService.java) | Aplicação (Serviço) | Reescrita completa do serviço com rolagem autoritativa, barreira de segurança de 0 HP, preservação de HP temporário no Descanso Longo e recuperação de cargas de talentos. |
| [`RulesEngineTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/RulesEngineTest.java) | Testes de Domínio | Teste TE-07, TE-05, TE-06 cobrindo fórmulas em SRD 2014 e 2024. |
| [`RestServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RestServiceTest.java) | Testes de Serviço | 9 testes unitários determinísticos (TP-01, TP-02, TE-01 a TE-06). |

---

## 📋 Resumo da Execução — Fase 3 (Controlador REST, DTOs & Testes de Integração)

Na Fase 3, expusemos os novos campos de exaustão e resumo de Dados de Vida na API HTTP REST, mapeamos respostas RFC 7807 `ProblemDetail` para falhas de domínio e validação, e validamos o ciclo completo de persistência via Testcontainers e MockMvc.

### Arquivos Criados e Modificados na Fase 3

| Arquivo | Camada | Modificação |
|---|---|---|
| [`RestValidationException.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/exception/RestValidationException.java) | Domínio (Exceção) | Nova exceção para violações de regras de descanso (sobregasto de dados de vida, progressão inexistente), herdando de `CharacterValidationException`. |
| [`GlobalExceptionHandler.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/core/web/GlobalExceptionHandler.java) | Web (Handler) | Tratamento especializado de `RestValidationException` mapeando para HTTP 400 Bad Request ProblemDetail. |
| [`CharacterClassSummaryDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterClassSummaryDto.java) | Aplicação (DTO) | Adicionados `classProgressionId`, `hitDie`, `hitDiceSpent`, `availableHitDice` mantendo compatibilidade com construtores anteriores. |
| [`CharacterSheetResult.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterSheetResult.java) | Aplicação (DTO) | Adicionado campo `exhaustion` com construtor sobrecarregado para compatibilidade. |
| [`CharacterSheetResponse.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterSheetResponse.java) | Web (DTO) | Adicionado campo `exhaustion` com construtor sobrecarregado para compatibilidade. |
| [`PlayerCharacterWebMapper.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/PlayerCharacterWebMapper.java) | Web (Mapper) | Mapeamento do campo `exhaustion` de `CharacterSheetResult` para `CharacterSheetResponse`. |
| [`CharacterSheetHydrator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydrator.java) | Aplicação (Serviço) | Hidratação de `classProgressionId`, `hitDie`, `hitDiceSpent`, `availableHitDice` e `exhaustion` no resultado da ficha. |
| [`CharacterQueryService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterQueryService.java) | Aplicação (Serviço) | Hidratação completa de `CharacterClassSummaryDto` na listagem de personagens. |
| [`PlayerCharacterPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterPersistenceAdapter.java) | Persistência | Correção de salvamento de `ability_scores` preservando ID em updates de personagem. |
| [`PlayerCharacterControllerTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterControllerTest.java) | Testes Web (MockMvc) | 4 testes cobrindo `POST /{id}/rest` com sucesso (TP-03), sobregasto de dados de vida retornando 400 (TE-02), tentativa de descanso em personagem morto retornando 422 (TE-03) e progressão inexistente retornando 400 (TE-04). |
| [`PlayerCharacterPersistenceAdapterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterPersistenceAdapterTest.java) | Testes de Integração (Testcontainers) | Teste TP-04 validando persistência real em PostgreSQL de `characters.exhaustion` e `class_progressions.hit_dice_spent` através de um ciclo de descanso. |
| [`tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/add-long-short-rest/tasks.md) | Documentação | Atualizadas tarefas da Fase 3 como concluídas `[x]`. |

---

## 🧪 Verificação e Resultados dos Testes — Fase 3

```bash
# Execução dos testes MockMvc do controlador:
mvn test -f player-character-service/backend/pom.xml "-Dtest=PlayerCharacterControllerTest"
# Resultado: Tests run: 20, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS

# Execução do teste de integração com PostgreSQL real (Testcontainers):
mvn test -f player-character-service/backend/pom.xml "-Dtest=PlayerCharacterPersistenceAdapterTest"
# Resultado: Tests run: 2, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS

# Execução da suíte completa de testes do backend:
mvn clean test -f player-character-service/backend/pom.xml
# Resultado: Tests run: 287, Failures: 0, Errors: 0, Skipped: 1 — BUILD SUCCESS
```

---

## 🎯 Contexto para a Fase 4 (Frontend UI/UX & Vitest)

Na **Fase 4**:
1. Atualizar tipos no frontend ([`dungeoneer-frontend/lib/api/characters.ts`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/lib/api/characters.ts)): adicionar `exhaustion`, `hitDie`, `hitDiceSpent`, `availableHitDice` em `CharacterClassSummary` e `RestPayload`.
2. Criar o componente de modal de descanso ([`dungeoneer-frontend/components/character-sheet/rest-modal.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/character-sheet/rest-modal.tsx)):
   - Seleção entre Descanso Curto e Descanso Longo.
   - Para Descanso Curto: seletores de Dados de Vida a gastar por classe (respeitando disponíveis), exibindo bônus de CON e estimativa.
   - Para Descanso Longo: confirmação detalhando recuperação total de PV, recuperação de metade dos dados de vida e redução de 1 de exaustão.
   - Validações: desabilitar botões se HP <= 0 ("Personagem Inconsciente / Morto").
3. Integrar na página da ficha ([`dungeoneer-frontend/app/characters/[id]/page.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/app/characters/[id]/page.tsx)):
   - Botões de "Descanso Curto" e "Descanso Longo" no cabeçalho/painel de status.
   - Contador de Dados de Vida disponíveis / totais na barra de status de combate.
   - Badge visual de Exaustão quando `exhaustion > 0`.
4. Implementar testes unitários Vitest (TP-05).
