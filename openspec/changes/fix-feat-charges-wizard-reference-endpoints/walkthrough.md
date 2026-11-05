# Walkthrough — Fix Feat Charges, Wizard Origin Filter & Full Security Hardening

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`  
> **Status:** in-progress  
> **Data:** 2026-11-04  

---

## Fase 1: Cargas de Talentos em Descanso & Filtro de Talentos de Origem no Wizard

### Resumo das Alterações
1. **Recálculo Dinâmico de Cargas & Regras por Ruleset (`player-character-service`):**
   - [`FeatChargeCalculator.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/FeatChargeCalculator.java): Adicionado suporte a `Ruleset` e resolução defensiva de recuperação. No `SRD_2024`, a Fúria ("Rage") recupera 1 carga fixa em descanso curto e todas as cargas no descanso longo.
   - [`CharacterFeat.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterFeat.java): Adicionado método sobrecarregado `recoverCharges(..., Ruleset ruleset)` preservando compatibilidade.
   - [`RestService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RestService.java): Em `recoverFeats` e `useFeat`, talentos legados com `maxCharges == null` têm seu teto máximo recalculado dinamicamente a partir dos efeitos mecânicos e são recuperados/utilizados sem exceções.
2. **Migração Flyway (`player-character-service`):**
   - [`V28__adapt_legacy_feat_charges_and_recovery.sql`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/resources/db/migration/V28__adapt_legacy_feat_charges_and_recovery.sql): Atualizados os registros `rage-2024` e `rage-2014` com metadados de recuperação e backfill das colunas `max_charges` e `current_charges` em `character_feats`.
3. **Filtro Estrito de Origem no Wizard (`dungeoneer-frontend`):**
   - [`page.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/app/characters/new/page.tsx): Atualizado para solicitar `featTypes: ["ORIGIN"]` na chamada para a API de referência e aplicar filtro estrito `f.featType === "ORIGIN"`, removendo a condição permissiva `!f.featType`.

---

### Verificação e Testes
- **Backend Unit Tests:**
  - `FeatChargeCalculatorTest`: 9 testes executados com sucesso (recuperação de Fúria no SRD 2024 vs SRD 2014, clamping, limites de recuperação).
  - `RestServiceTest`: 9 testes executados com sucesso (descanso curto com Fúria 2024 recuperando 1 carga, recálculo dinâmico de `maxCharges` em fichas legadas, uso de carga com `useFeat`).
  - Total: 18 testes unitários passando (`mvn -f player-character-service/backend/pom.xml test "-Dtest=FeatChargeCalculatorTest,RestServiceTest"`).
- **Frontend Unit Tests:**
  - `character-wizard.test.tsx`: 2 testes executados com sucesso (`vitest run __tests__/character-wizard.test.tsx`), confirmando o envio do parâmetro `featTypes=ORIGIN` e a filtragem estrita no wizard.

---

### Portão de Pausa & Próximos Passos
A Fase 1 está 100% concluída e validada.  
O usuário deve executar o commit da Fase 1 antes de prosseguirmos para a Fase 2:
```bash
git add player-character-service/ dungeoneer-frontend/ openspec/changes/fix-feat-charges-wizard-reference-endpoints/
git commit -m "feat(character): fix feat charges recovery on rest and enforce wizard origin filter"
```
