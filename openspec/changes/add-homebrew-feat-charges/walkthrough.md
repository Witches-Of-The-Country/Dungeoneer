# Walkthrough: Add Homebrew Feat Charges

> **Change ID:** `add-homebrew-feat-charges`  
> **Status:** Em Progresso (Fase 1 Concluída)  
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

## 🧪 Verificação e Resultados dos Testes

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

## 🎯 Contexto para a Fase 2 (Aplicação, Hidratação, DTOs e Integração)

Na **Fase 2**:
1. Atualizar [`CharacterSheetHydrator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterSheetHydrator.java) para usar o `FeatChargeCalculator.calculateMaxCharges` na hidratação da ficha.
2. Atualizar [`CreateCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java) e [`LevelUpService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java) para inicializar cargas em talentos adquiridos.
3. Atualizar DTOs (`CreateHomebrewFeatRequest`, `FeatDto`, `CharacterFeatDto`) com anotações Jakarta Validation e campos de configuração de cargas.
4. Adicionar validação cruzada em [`HomebrewService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java) (ex.: exigir `chargeAbility` quando a fórmula for `ABILITY_MODIFIER`) retornando RFC 7807 400 ProblemDetail.
5. Criar testes MockMvc e de integração (cenários TP-03, TE-03, TE-04).
