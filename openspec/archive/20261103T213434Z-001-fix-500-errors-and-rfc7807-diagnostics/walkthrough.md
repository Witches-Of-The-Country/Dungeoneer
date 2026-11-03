# Walkthrough — 001: Correção de Erros 500 na Criação de Personagem e Itens e Diagnósticos RFC 7807

## Resumo do Progresso por Fase

---

### Fase 1: Serialização JSONB de Talentos e Deduplicação na Cascata de Persistência
- **Status:** Concluído ✅
- **Arquivos Modificados:**
  - `CharacterFeatJpaEntity.java`: Adicionado sanitizador defensivo `sanitizeChoicesJson` nos construtores e setters, garantindo que qualquer entrada que não seja JSON estruturado seja formatada como `{"source":"..."}` antes da gravação na coluna `JSONB` do PostgreSQL.
  - `CreateCharacterService.java`: 
    - Formatação de `choicesJson` como JSON válido (`{"source":"..."}`).
    - Deduplicação de talentos iniciais (`initialCharacterFeats`) via `Set<UUID>`.
    - Remoção do loop redundante `saveCharacterFeat(cf)` após `playerCharacterRepository.save(character)`, confiando na cascata JPA `CascadeType.ALL` e eliminando violações de unicidade (`uq_character_feat`).
  - `LevelUpService.java`: Garantido que o destravamento de talentos de classe, subclasse e escolhas de ASI inicializem `choicesJson` com JSON estruturado válido.
  - `CreateCharacterServiceTest.java`: Adicionado teste `shouldCreateCharacterWithOriginFeatAndValidJsonChoices` cobrindo o cenário exato de falha (nível 5 com Origin Feat).
- **Validação Local:**
  - `mvn -f player-character-service/backend/pom.xml test -Dtest=CreateCharacterServiceTest,LevelUpServiceTest` (14/14 testes aprovados com sucesso).

---

### Fase 2: DTO de Borda para Propriedades Mecânicas de Itens e Discriminador JSONB
- **Status:** Concluído ✅
- **Arquivos Criados/Modificados:**
  - `HomebrewDamageRollRequest.java`: DTO de borda para representar dados de dano na criação de itens.
  - `HomebrewItemMechanicsRequest.java`: DTO de borda com método `toDomain()` mapeando limpo para a hierarquia de domínio (`WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, `EmptyItemMechanics`) sem poluir o domínio com anotações de framework.
  - `CreateHomebrewItemRequest.java`: Atualizado para receber `HomebrewItemMechanicsRequest` em vez da interface de domínio selada `ItemMechanics`.
  - `HomebrewService.java`: Usa `request.resolveMechanicalProperties()` para alimentar a entidade de domínio.
  - `ItemJpaEntity.java`: 
    - `serializeMechanics()` injeta explicitamente o discriminador `"type"` (`"WEAPON"`, `"ARMOR"`, `"CONSUMABLE"`).
    - `deserializeMechanics()` faz fallback defensivo inteligente inferindo o tipo caso registros legados não possuam `"type"`.
  - `HomebrewServiceTest.java`: Adicionados testes de mapeamento e round-trip JSONB para armas e fallback de registros legados.
  - `ReferenceDataControllerTest.java`: Adicionado teste de integração web verificando `POST /api/reference/items` com `mechanicalProperties` retornando HTTP 201 Created.
- **Validação Local:**
  - `mvn -f player-character-service/backend/pom.xml test -Dtest=HomebrewServiceTest,ReferenceDataControllerTest` (15/15 testes aprovados com sucesso).

---

### Fase 3: Tratamento de Exceções e Diagnósticos RFC 7807 no Backend
- **Status:** Concluído ✅
- **Arquivos Modificados:**
  - `GlobalExceptionHandler.java`:
    - Adicionado `@ExceptionHandler(DataIntegrityViolationException.class)` retornando HTTP 409 Conflict com mensagens explicativas em pt-BR (conflito de unicidade/chaves duplicadas, chaves estrangeiras inexistentes, erros de sintaxe JSON) e `instance` apontando para a URI requisitada.
    - Adicionado `@ExceptionHandler(Exception.class)` como fallback global retornando HTTP 500 com formato RFC 7807 `ProblemDetail`, título em pt-BR e mensagem descritiva sem vazar credenciais ou detalhes internos.
  - `GlobalExceptionHandlerTest.java`: Adicionados testes unitários validando respostas RFC 7807 para violação de integridade (409) e falha genérica do servidor (500).
- **Validação Local:**
  - `mvn -f player-character-service/backend/pom.xml test -Dtest=GlobalExceptionHandlerTest` (6/6 testes aprovados com sucesso).

---

### Fase 4: Exibição de Diagnósticos RFC 7807 e Formatação de Toasts no Frontend
- **Status:** Concluído ✅
- **Arquivos Modificados:**
  - `client.ts`:
    - Adicionado suporte à extração de payload de erro Spring padrão (`error`, `path`) e tradução contextual para pt-BR amigável quando o backend ainda não retornar RFC 7807 customizado.
    - Suporte a mensagens personalizadas para conflito de unicidade de talentos (`uq_character_feat`) e caminhos de API com falha interna.
  - `client.test.ts`:
    - Adicionados testes unitários verificando a formatação de payloads de erro do Spring Boot 500, restrições únicas e falhas de requisição no `apiFetch`.
- **Validação Local:**
  - `npm --prefix dungeoneer-frontend run test:unit` (16 arquivos de teste aprovados, 51/51 testes passando).

---

### Fase 5: Verificação Completa e Finalização da Spec 001
- **Status:** Concluído ✅
- **Validação Executada:**
  - `mvn -f player-character-service/backend/pom.xml test -Dtest="!*PersistenceAdapterTest"`: 186 testes unitários e de integração web aprovados sem nenhuma falha (`BUILD SUCCESS`).
  - `npm --prefix dungeoneer-frontend run test:unit`: 16 suítes de teste (51/51 testes) aprovadas com sucesso no Vitest.
- **Tarefas Finalizadas:**
  - Todas as tarefas de implementação, testes, documentação e review gates de `tasks.md` marcadas como `[x]`.
  - Critérios de aceitação da proposta `001-fix-500-errors-and-rfc7807-diagnostics` verificados e aprovados.

---

## Conclusão da Spec 001
A mudança `001-fix-500-errors-and-rfc7807-diagnostics` está 100% implementada, testada e em conformidade estrita com o SDD Stack Contract e Clean Architecture.

