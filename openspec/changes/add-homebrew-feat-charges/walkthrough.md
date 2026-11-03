# Walkthrough: Add Homebrew Feat Charges

> **Change ID:** `add-homebrew-feat-charges`  
> **Status:** Em Progresso (Fases 1 e 2 Concluídas)  
> **Data:** 2026-11-03  

---

## 📋 Resumo da Execução — Fase 1 (Domínio, Fórmulas de Cargas e Testes Unitários)

Nesta primeira fase, implementamos as fundações do sistema de cargas para talentos homebrew na camada de Domínio, seguindo rigorosamente os princípios da Clean Architecture (zero dependências de framework, zero imports de Spring/Lombok/Jackson na camada de domínio).

### 1. Arquivos Criados e Modificados

| Arquivo | Camada | Modificação |
|---|---|---|
| [`ChargeType.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ChargeType.java) | Domínio | Novo enum definindo fórmulas de máximo de cargas (`FIXED`, `ABILITY_MODIFIER`, `PROFICIENCY`, `LEVEL`). |
| [`ChargeRecoveryType.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ChargeRecoveryType.java) | Domínio | Novo enum definindo regras de recuperação por descanso (`NONE`, `ALL`, `FIXED`, `MODIFIER`, `PROFICIENCY`, `LEVEL`). |
| [`MechanicalEffects.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffects.java) | Domínio | Extensão do record para incluir campos estruturados de cargas (`maxChargesType`, `chargeAbility`, `fixedMaxCharges`, `shortRestRecoveryType`, `shortRestRecoveryFixed`, `longRestRecoveryType`, `longRestRecoveryFixed`) e validações defensivas. |
| [`FeatChargeCalculator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/FeatChargeCalculator.java) | Domínio | Utilitário puro de domínio para cálculo dinâmico do máximo de cargas e montante de recuperação com clamping contra overflow e suporte a fallback de legado. |
| [`CharacterFeat.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterFeat.java) | Domínio | Métodos `withMaxCharges` (com clamping de `currentCharges` caso o teto reduza), `recoverCharges` com avaliação independente por tipo de descanso e clamping de limites. |
| [`FeatChargeCalculatorTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/FeatChargeCalculatorTest.java) | Testes Unitários | 8 testes unitários cobrindo TP-01, TP-02, TE-01, TE-02, fallbacks de legado e clampings. |
| [`CharacterFeatTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/CharacterFeatTest.java) | Testes Unitários | 6 testes unitários cobrindo TE-02, TE-05, clamping de teto e validações de limites. |
| [`MechanicalEffectsTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffectsTest.java) | Testes Unitários | Testes de integridade, normalização de atributo e validações contra números negativos. |
| [`tasks.md`](file:///D:/Projects/Dungeoneer/openspec/changes/add-homebrew-feat-charges/tasks.md) | Documentação | Atualizado checklist de tarefas concluídas. |

---

## 🧪 Verificação e Resultados dos Testes — Fase 1

Executada a suíte de testes unitários e a suíte completa de regressão do microsserviço:

```bash
# 1. Testes unitários focados nas novas regras de domínio:
mvn test -f player-character-service/backend/pom.xml "-Dtest=FeatChargeCalculatorTest,CharacterFeatTest,MechanicalEffectsTest"
# Resultado: Tests run: 19, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS

# 2. Suíte de regressão completa do player-character-service (incluindo Testcontainers):
mvn test -f player-character-service/backend/pom.xml
# Resultado: Tests run: 252, Failures: 0, Errors: 0, Skipped: 1 — BUILD SUCCESS
```

---

## 📋 Resumo da Execução — Fase 2 (Serviços de Aplicação, Hidratação, DTOs e Integração)

Na Fase 2, integramos o cálculo dinâmico de cargas nos fluxos de aplicação e hidratação da ficha de personagem, garantindo persistência compatível em JSONB e validações defensivas RFC 7807 na criação de talentos homebrew.

### 1. Arquivos Modificados e Testes Criados

| Arquivo | Camada | Modificação |
|---|---|---|
| [`CharacterSheetHydrator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydrator.java) | Aplicação | Recalcula dinamicamente `maxCharges` usando `FeatChargeCalculator` e aplica clamping a `currentCharges` em cada hidratação para a lista de talentos e ações. |
| [`CharacterFeatDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterFeatDto.java) | Aplicação DTO | Sobrecarga de `fromDomain` recebendo `currentCharges` e `maxCharges` dinâmicos calculados pelo hydrator. |
| [`CreateCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java) | Aplicação | Inicializa cargas de talentos concedidos na criação via `FeatChargeCalculator.calculateMaxCharges`. |
| [`LevelUpService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java) | Aplicação | Inicializa cargas de novos talentos (arquétipo, subclasse, linhagem e talentos bônus) via `FeatChargeCalculator.calculateMaxCharges`. |
| [`HomebrewService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java) | Aplicação | Implementada validação defensiva cruzada `validateFeatMechanicalEffects` (exige `chargeAbility` quando a fórmula for `ABILITY_MODIFIER`, valida números fixos positivos), lançando `CharacterValidationException` (RFC 7807 ProblemDetail). |
| [`CharacterSheetHydratorTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydratorTest.java) | Testes Aplicação | Teste cobrindo recálculo dinâmico baseado em atributo do personagem e clamping de cargas excedentes. |
| [`HomebrewServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/HomebrewServiceTest.java) | Testes Aplicação | Testes unitários para TP-03 (criação e persistência de talento com cargas) e TE-03 (rejeição de configurações inválidas). |
| [`HomebrewCreationControllerTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/HomebrewCreationControllerTest.java) | Testes Web MockMvc | Testes de API para TP-03 (POST 201 Created), TE-03 (enum malformado 400 Bad Request, validação 422 ProblemDetail). |
| [`MechanicalEffectsTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/MechanicalEffectsTest.java) | Testes Persistência | Testes para TE-04 (desserialização segura de JSON legados sem novos campos) e TP-03 (round-trip JSONB completo). |

---

## 🧪 Verificação e Resultados dos Testes — Fase 2

```bash
# 1. Testes direcionados da Fase 2:
mvn test -f player-character-service/backend/pom.xml "-Dtest=CharacterSheetHydratorTest,HomebrewServiceTest,HomebrewCreationControllerTest,MechanicalEffectsTest"
# Resultado: Tests run: 37, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS

# 2. Suíte de regressão completa do player-character-service (incluindo Testcontainers):
mvn test -f player-character-service/backend/pom.xml
# Resultado: Tests run: 262, Failures: 0, Errors: 0, Skipped: 1 — BUILD SUCCESS
```

---

## 🎯 Contexto para a Fase 3 (Frontend UI, Diálogo de Criação, Feats Tab e Vitest)

Na **Fase 3**:
1. Atualizar interfaces TypeScript em `dungeoneer-frontend/lib/api/homebrew.ts` e `dungeoneer-frontend/lib/api/characters.ts`.
2. Adicionar construtor interativo de cargas em `create-homebrew-dialog.tsx` (fórmula de cargas, atributo, regras independentes de descanso curto e longo).
3. Atualizar renderização de talentos em `feats-tab.tsx` com contadores de cargas e badges de recuperação.
4. Testes Vitest (TP-04).
5. Preenchimento final dos arquivos delta em `openspec/changes/add-homebrew-feat-charges/specs/`.
