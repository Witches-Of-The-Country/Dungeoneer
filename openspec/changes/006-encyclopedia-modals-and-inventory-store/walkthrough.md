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

## Handoff para a Fase 2:
- **Próxima etapa**: Fase 2 — Implementar endpoints de atualização de Homebrew (`PUT /api/reference/{category}/{id}`) com validação estrita de segurança (`ruleset == HOMEBREW`).
