# Walkthrough: level-up-half-feat-ability-choice

## Fase 1: Alinhamento de DTOs, Validação no LevelUpService e Testes de Unidade

### 1. Arquivos Modificados
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Srd2014RulesEngine.java`:
  - Expandido o método `normalizeAbility` para aceitar abreviações e nomes em português (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`, além de variações acentuadas como `FORÇA`, `DESTREZA`, `SABEDORIA`, etc.), garantindo interoperabilidade entre UI e backend.
- `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java`:
  - Implementada a validação de half-feats (`abilityBonusChoice` e `abilityBonuses`) ao subir de nível com `command.selectedFeatId() != null`.
  - Valida se o cliente forneceu exatamente a quantidade `chooseCount` de atributos contidos em `allowedAbilities` com valor igual a `bonusAmount` (+1).
  - Rejeita com `CharacterValidationException` (HTTP 422):
    - Seleção de half-feat sem envio do atributo escolhido;
    - Atributo fora da lista de permitidos do talento;
    - Valor de bônus superior ao estipulado (+1);
    - Envio de bônus de atributo quando o talento selecionado não for um half-feat.
  - Aplica com segurança os bônus aos `AbilityScores` do personagem, respeitando o teto de 20.
  - Tratamento defensivo de null pointer em `usesPerRest`.
- `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/LevelUpServiceTest.java`:
  - Adicionados 5 novos testes de unidade cobrindo o caminho feliz (+1 em Força em talento Atleta) e os 4 modos de falha/borda (sem atributo, atributo não permitido, bônus com valor inválido, incremento indevido em talento comum).

### 2. Resultados de Verificação e Testes
- **Comando Executado**: `mvn -f player-character-service/backend/pom.xml test`
- **Resultado**: `BUILD SUCCESS` (01:24 min)
- **Sumário**: 236 testes executados (5 novos testes), 0 falhas, 0 erros, 1 ignorado.

### 3. Contexto de Transição para a Fase 2
- A Fase 1 está 100% concluída e testada.

---

## Fase 2: Interface no Frontend (LevelUpWizardModal) e Testes de Unidade

### 1. Arquivos Modificados
- `dungeoneer-frontend/lib/api/encyclopedia.ts`:
  - Enriquecida a tipagem TypeScript para `Feat` e `MechanicalEffects`, adicionando suporte tipado a `AbilityBonusChoice`, `prerequisiteRules` e `spellChoice`.
- `dungeoneer-frontend/components/character-sheet/level-up-wizard-modal.tsx`:
  - Adicionado `halfFeatChoiceConfig` derivado do talento selecionado no Passo 3.
  - Implementado sub-painel interativo `"Bônus de Atributo do Talento (+1)"` exibindo chips/botões seletores para os atributos permitidos (`allowedAbilities`), com cálculo da pontuação atual e projetada (+1, limitado ao teto de 20).
  - Bloqueio no botão "Avançar" (`handleAdvanceStep`) caso o jogador selecione um half-feat sem escolher o atributo a ser incrementado, exibindo toast explicativo.
  - Atualização do resumo do Passo 4 com o nome do talento e o atributo selecionado (ex: `Talento: Atleta (+1 em Força)`).
  - Inclusão do atributo escolhido no payload da mutação `levelUpMutation.mutate` como `abilityScoreIncreases: { [chosenStat]: 1 }`.
- `dungeoneer-frontend/__tests__/level-up-wizard.test.tsx`:
  - Adicionado mock de talento half-feat (*Atleta*, com `abilityBonusChoice` permitindo `STR` ou `DEX`).
  - Adicionado novo teste de unidade cobrindo:
    1. Renderização do seletor de atributos ao clicar no card de um half-feat;
    2. Bloqueio de avanço para o Passo 4 sem seleção de atributo;
    3. Seleção de Força (+1 Força: 15 → 16);
    4. Exibição correta no resumo final;
    5. Confirmação do envio de `abilityScoreIncreases: { STR: 1 }` na mutação `useLevelUp`.

### 2. Resultados de Verificação e Testes
- **Comando Executado**: `npm --prefix dungeoneer-frontend run test:unit`
- **Resultado**: `19 passed (19 test files)`, `71 passed (71 tests)` (28.95s).

### 3. Contexto de Transição para a Fase 3
- A Fase 2 está 100% concluída e testada.
- **Próximos Passos (Fase 3)**: Verificação & Integração Ponta a Ponta:
  - Execução da suíte completa de testes backend (`mvn test`).
  - Execução de build de produção do frontend (`npm run build`).
  - Rebuild e reinício do contêiner Docker `player-character-service`.
  - Validação funcional ao vivo e encerramento/arquivamento da spec.
