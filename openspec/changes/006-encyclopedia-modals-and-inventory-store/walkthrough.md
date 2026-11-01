# Registro de Execução: Spec 006 — Modais da Enciclopédia, Edição Homebrew e Loja do Inventário

## Fase 1: Correções de Domínio e Hidratação no Backend (Concluída)

### Arquivos Modificados / Criados:
- [`PlayerCharacter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java): Preserva `currencyPouch` e `avatarUrl` nas mutações imutáveis `withItems`, `withFeats` e `withLevelUp` (eliminando o bug de moedas zerando).
- [`CharacterFeatDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterFeatDto.java): Adicionado campo `source` (origem do talento).
- [`CharacterClassSummaryDto.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterClassSummaryDto.java): Criado DTO com `className`, `subclassName` e `classLevel`.
- [`CharacterSummaryResult.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterSummaryResult.java) & [`CharacterSummaryResponse.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterSummaryResponse.java): Adicionados `subclassName` e lista `classes`.
- [`PlayerCharacterWebMapper.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/PlayerCharacterWebMapper.java): Mapeia subclasse e lista de classes.
- [`CharacterQueryService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterQueryService.java): Injeta e invoca `CharacterSheetHydrator.hydrate(...)` para carregar itens e talentos reais; enriquece `listCharacters` com subclasse principal e multiclasses.
- [`UpdateCharacterService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/UpdateCharacterService.java): Injeta e invoca `CharacterSheetHydrator.hydrate(...)`.
- [`PlayerCharacterTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacterTest.java): Testes cobrindo retenção de `currencyPouch` e `avatarUrl` em `withItems`, `withFeats`, `withLevelUp`.
- [`CharacterQueryServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterQueryServiceTest.java): Testes cobrindo hidratação e resumo multiclasse com subclasses.
- [`UpdateAndDeleteCharacterServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/UpdateAndDeleteCharacterServiceTest.java): Atualizado para o novo construtor com o hydrator.

### Validação dos Testes:
```bash
mvn -f player-character-service/backend/pom.xml test -Dtest="PlayerCharacterTest,CharacterQueryServiceTest,UpdateAndDeleteCharacterServiceTest"
# Resultado: Tests run: 32, Failures: 0, Errors: 0, Skipped: 0 (BUILD SUCCESS)
```

---

## Fase 2: Endpoints de Atualização de Homebrew & Checagens de Segurança (Concluída)

### Arquivos Modificados:
- [`CreateHomebrewUseCase.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/CreateHomebrewUseCase.java): Declarados métodos de atualização para magias, talentos, itens, arquétipos, linhagens e subclasses.
- [`HomebrewService.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/HomebrewService.java): Implementados os 6 métodos de atualização com barreira de segurança estrita: mutações são permitidas exclusivamente quando `ruleset == Ruleset.HOMEBREW`. Tentativas de editar conteúdo SRD lançam `CharacterValidationException`.
- [`ReferenceDataRepository.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java) & [`ReferenceDataPersistenceAdapter.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java): Adicionado `findSpellById(UUID id)`.
- [`SpellJpaEntity.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpellJpaEntity.java): Implementado método de conversão `toDomain()`.
- [`ReferenceDataController.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java): Expostos endpoints `PUT /api/reference/{spells,feats,items,classes,lineages,subclasses}/{id}` com validação `@Valid`.
- [`HomebrewServiceTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/HomebrewServiceTest.java): Testes cobrindo atualizações de sucesso e rejeição defensiva de itens/magias SRD.
- [`ReferenceDataControllerTest.java`](file:///D:/Projects/Dungeoneer/player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataControllerTest.java): Testes de integração WebMvc para `PUT /api/reference/items/{id}` (200 OK e 422 na tentativa de mutação de SRD).

### Validação dos Testes & Container:
```bash
mvn -f player-character-service/backend/pom.xml test -Dtest="HomebrewServiceTest,ReferenceDataControllerTest"
# Resultado: Tests run: 24, Failures: 0, Errors: 0, Skipped: 0 (BUILD SUCCESS)
docker --context default compose up -d --build player-character-service
# Resultado: Container recriado e online em 11s.
```

---

## Fase 3: Correção de Layout e Tamanho dos Inputs de Moedas (Concluída)

### Arquivos Modificados:
- [`create-homebrew-dialog.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx):
  - `Raridade` e `Peso` foram isolados em um grid de 2 colunas.
  - `Custo (Moedas)` ganhou uma linha exclusiva com fundo destacado, 5 colunas responsivas, largura confortável por moeda e classes utilitárias `[appearance:textfield] [&::-webkit-outer-spin-button]:appearance-none [&::-webkit-inner-spin-button]:appearance-none` para remover botões de spinner do browser que bloqueavam a digitação.
  - O valor zero inicial foi ajustado para placeholder visual `0`, permitindo digitação direta de valores com múltiplos dígitos (como `50` ou `1000`) sem cortes ou sobreposição.

### Validação dos Testes:
```bash
npm --prefix dungeoneer-frontend run test:unit
# Resultado: 18 test files passed (61 tests passed)
```

---

## Fase 4: Ficha de Talentos, Modais e Card Multiclasse (Concluída)

### Arquivos Modificados / Criados:
- [`characters.ts`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/lib/api/characters.ts): Adicionado `CharacterClassSummary` e campos `subclassName` e `classes` ao `CharacterSummary`.
- [`page.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/app/characters/page.tsx): Atualizado o card de personagens para renderizar a classe principal com sua subclasse e todas as multiclasses secundárias com seus níveis (ex: `Nível 7 • Guerreiro (Mestre da Batalha) 5 / Mago 2 • Humano`).
- [`feat-detail-modal.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/encyclopedia/feat-detail-modal.tsx): Criado modal completo de detalhes de talentos com exibição de lore, badges de origem (Classe, Subclasse, Linhagem, Origem, Geral, Homebrew), pré-requisitos, mecânicas ativas (bônus de atributos, CA, ações e cargas) e suporte ao botão "Editar" para itens Homebrew.
- [`feats-tab.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/character-sheet/feats-tab.tsx): Integrado `FeatDetailModal` ao clicar em qualquer card de talento, com badges de origem e indicação visual.
- [`currency-pouch-card.tsx`](file:///D:/Projects/Dungeoneer/dungeoneer-frontend/components/character-sheet/currency-pouch-card.tsx): Ajustados inputs do modo de edição de moedas para ocultar setas de incremento do navegador e evitar o zero preso à esquerda.

### Validação dos Testes:
```bash
npm --prefix dungeoneer-frontend run test:unit
# Resultado: 18 test files passed (61 tests passed)
```

---

## Handoff para a Fase 5:
- **Próxima etapa**: Fase 5 — Criação dos modais de detalhes da Enciclopédia (`SpellDetailModal`, `ItemDetailModal`, `LineageDetailModal`) e integração nos cards de referência das páginas da enciclopédia com o botão "Editar" para Homebrews.
