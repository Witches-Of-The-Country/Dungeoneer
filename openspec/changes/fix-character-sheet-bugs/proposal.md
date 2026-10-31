# Proposal: Fix Character Sheet Bugs

> **Change ID:** `fix-character-sheet-bugs`
> **Status:** draft
> **Author:** @aluno
> **Created:** 2026-10-30
> **Last Updated:** 2026-10-30

---

## Summary

Este change corrige quatro bugs identificados no frontend da ficha de personagem do Dungeoneer. Os bugs afetam a experiência de criação e edição de personagens: rolagem de vida sendo permitida antes da escolha de classe (causando estado inválido), conteúdo SRD exibido em inglês ao invés de PT-BR, notas do personagem sobrescrevendo umas às outras em vez de serem incrementais, e atraso visível no drag & drop de itens em relação ao cursor do mouse.

## Motivation

- **Bug 1 — Rolagem de vida prematura:** Permitir que o usuário role os dados de vida antes de selecionar uma classe resulta em estado corrompido, pois o dado de vida (`hit_die`) é uma propriedade da classe. O sistema não sabe qual dado usar.
- **Bug 2 — Conteúdo SRD em inglês:** O Dungeoneer tem PT-BR como idioma da UI. Todo conteúdo de referência SRD (nomes de classes, raças, magias, habilidades, etc.) deve ser exibido em português.
- **Bug 3 — Notas não incrementais:** O campo de notas do personagem funciona como um único bloco de texto. O esperado é que o usuário possa adicionar notas individuais com título e corpo, acumulando-as ao longo do tempo.
- **Bug 4 — Lag no drag & drop:** O elemento arrastado fica visivelmente atrasado em relação ao cursor do mouse durante movimentos rápidos, degradando a usabilidade da ficha.

## Scope

### In Scope
- Bloquear o botão/ação de rolagem de vida até que uma classe seja selecionada no wizard de criação de personagem
- Traduzir para PT-BR todos os textos SRD visíveis na UI (classes, raças/linhagens, backgrounds, magias, habilidades)
- Refatorar o componente de notas para suportar múltiplas notas com título, corpo e timestamp
- Criar endpoint/persistência backend para lista de notas incrementais (se necessário)
- Corrigir a implementação de drag & drop para eliminar o lag entre cursor e elemento arrastado

### Out of Scope
- Tradução de textos de sistema (erros, labels de formulário) — já cobertos por outro change
- Adição de novas funcionalidades às notas (edição inline, categorias, tags)
- Refatoração completa do wizard de criação de personagem
- Implementação de homebrew ou tradução de conteúdo não-SRD

## Proposed Solution

### Approach

**Bug 1 — Rolagem de vida após classe:** No wizard de criação, o botão de rolar vida deve ser desabilitado (`disabled`) e exibir tooltip explicativo enquanto `character.class` for `null` ou `undefined`. A validação deve existir tanto no frontend (UI guard) quanto no backend (validação no use case de atualização de HP).

**Bug 2 — Tradução SRD PT-BR:** Criar um mapa de tradução (`srd-translations.ts`) no frontend que mapeia os identificadores/slugs SRD para seus equivalentes PT-BR. Todas as queries TanStack que buscam dados de referência devem aplicar esse mapa antes de renderizar. No backend, avaliar se os dados SRD em banco já contêm campo `name_pt_br` ou se precisam de migration.

**Bug 3 — Notas incrementais:** Refatorar o modelo de notas de `notes: string` para `notes: CharacterNote[]` onde `CharacterNote = { id, title, body, createdAt }`. No frontend, criar um componente `NoteCard` e `AddNoteForm`. No backend, atualizar o schema PostgreSQL com uma tabela `character_note` e criar os endpoints REST correspondentes.

**Bug 4 — Drag & drop lag:** O lag ocorre tipicamente por uso de `onDrag` em vez de `onDragOver`, ou por transforms CSS aplicados via `setState` (re-render lento). A solução é migrar para manipulação direta do DOM via `ref` durante o drag, ou adotar a biblioteca `@dnd-kit/core` que usa `pointer events` e `transform` sem re-render, eliminando o lag.

### Impacted Files

| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/app/characters/new/page.tsx` | Modify | **Bug 1 & 4**: Desabilitar botão de rolagem de HP sem classe; corrigir lag no DnD removendo `transition-all` durante drag do `DraggableScoreChip` |
| `dungeoneer-frontend/lib/srd-translations.ts` | Create | **Bug 2**: Mapeamentos de classes, linhagens, antecedentes, talentos e perícias para PT-BR com função `translateSrd()` |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | **Bug 2 & 3**: Tipos e hooks de `CharacterNote` (`useCharacterNotes`, `useAddCharacterNote`); transformações PT-BR nas queries |
| `dungeoneer-frontend/app/characters/page.tsx` | Modify | **Bug 2**: Renderização de nomes traduzidos nos cards de personagens |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Modify | **Bug 2 & 3**: Tradução da ficha; substituição de textarea único por `<AddNoteForm>` e lista de `<NoteCard>` |
| `dungeoneer-frontend/app/characters/[id]/edit/page.tsx` | Modify | **Bug 2 & 3**: Remoção do campo único de notas; traduções PT-BR |
| `dungeoneer-frontend/components/note-card.tsx` | Create | **Bug 3**: Componente para exibir nota individual (título, corpo, data formatada PT-BR) |
| `dungeoneer-frontend/components/add-note-form.tsx` | Create | **Bug 3**: Formulário de adição de nota com validação e mutação React Query |
| `dungeoneer-frontend/__tests__/wizard-ability-roll-guard.test.tsx` | Modify | **Bug 1**: Testes unitários para guard de rolagem de vida sem classe selecionada |
| `dungeoneer-frontend/__tests__/character-sheet.test.tsx` | Modify | **Bug 3**: Atualização de testes para adição incremental de notas |
| `dungeoneer-frontend/__tests__/character-list-and-edit.test.tsx` | Modify | **Bug 2 & 3**: Verificação de campos traduzidos e comportamento de edição |
| `player-character-service/backend/src/main/resources/db/migration/V5__create_character_note_table.sql` | Create | **Bug 3**: Flyway migration criando `character_notes` e migrando dados legados de `notes` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/CharacterNote.java` | Create | **Bug 3**: Entidade pura de domínio Clean Architecture com validação de invariantes |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java` | Modify | **Bug 3**: Deprecação de campo `notes` legado |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/CharacterNoteRepositoryPort.java` | Create | **Bug 3**: Port de saída do repositório de notas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/AddCharacterNoteUseCase.java` | Create | **Bug 3**: Port de entrada para criação de notas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/ListCharacterNotesUseCase.java` | Create | **Bug 3**: Port de entrada para listagem de notas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/AddCharacterNoteCommand.java` | Create | **Bug 3**: Command record da camada de aplicação |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CharacterNoteResult.java` | Create | **Bug 3**: Result record da camada de aplicação |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteService.java` | Create | **Bug 3**: Implementação dos use cases com validação de ownership e persistência |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/RollService.java` | Modify | **Bug 1**: Validação explícita de `archetypeId != null` em `rollHitPoints` com `CharacterValidationException` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CreateCharacterNoteRequest.java` | Create | **Bug 3**: REST DTO com `@NotBlank`, `@Size` para título e corpo |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/dto/CharacterNoteResponse.java` | Create | **Bug 3**: REST response record |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java` | Modify | **Bug 3**: Endpoints `POST /api/characters/{id}/notes` e `GET /api/characters/{id}/notes` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/PlayerCharacterWebMapper.java` | Modify | **Bug 3**: Mapeadores para requisições, resultados e respostas de notas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNoteJpaEntity.java` | Create | **Bug 3**: Entidade JPA mapeada para `player_character.character_notes` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/SpringDataCharacterNoteRepository.java` | Create | **Bug 3**: Interface Spring Data JPA |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterNotePersistenceAdapter.java` | Create | **Bug 3**: Adaptador de persistência para `CharacterNoteRepositoryPort` |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/RollServiceTest.java` | Modify | **Bug 1**: Teste unitário rejeitando arquétipo nulo na rolagem de vida |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteServiceTest.java` | Create | **Bug 3**: Teste unitário para `AddCharacterNoteUseCase` e `ListCharacterNotesUseCase` |
| `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterControllerTest.java` | Modify | **Bug 1 & 3**: Testes de integração para endpoints de notas e validação de rolagem de HP sem classe |

### Data Model Changes (if applicable)

**Notas incrementais** requerem nova tabela no schema `player_character`:
```sql
CREATE TABLE character_note (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  character_id UUID NOT NULL REFERENCES player_character(id) ON DELETE CASCADE,
  title       VARCHAR(255) NOT NULL,
  body        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```
O campo `notes TEXT` na tabela `player_character` deve ser deprecado após migração.

## Acceptance Criteria

- [ ] **Given** o wizard de criação sem classe selecionada **When** o usuário tenta rolar vida **Then** o botão está desabilitado e exibe mensagem "Selecione uma classe primeiro"
- [ ] **Given** classe selecionada no wizard **When** o usuário clica em "Rolar Vida" **Then** a rolagem usa o hit_die correto da classe escolhida
- [ ] **Given** qualquer tela com listagem de classes, raças ou magias **When** o conteúdo é exibido **Then** todos os nomes e descrições aparecem em PT-BR
- [ ] **Given** a seção de notas da ficha **When** o usuário adiciona uma nota com título e corpo **Then** a nota é salva individualmente e exibida em lista cronológica sem sobrescrever notas anteriores
- [ ] **Given** a ficha com múltiplas notas **When** o usuário visualiza a seção de notas **Then** cada nota exibe seu título, corpo e data de criação
- [ ] **Given** qualquer tela com drag & drop **When** o usuário arrasta um elemento rapidamente **Then** o elemento segue o cursor sem atraso perceptível (< 16ms de lag)

## Defensive Security & Validation Considerations

### Input Validation
- Título e corpo das notas devem ter limites de tamanho (ex: título ≤ 255 chars, corpo ≤ 10.000 chars) com validação Jakarta no DTO.
- O `character_id` nas notas deve ser validado como UUID válido pertencente ao personagem correto.

### Secret Isolation
- N/A — nenhuma credencial ou chave é manipulada por este change.

### Authorization & Authentication Impact
- N/A (MVP sem autenticação). Os endpoints de notas seguem a mesma política permissiva atual.

### Other Security Risks
- Garantir que o corpo das notas seja sanitizado no frontend antes de renderização para prevenir XSS (usar `textContent` em vez de `innerHTML`).

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|--------------------|
| TP-01 | Rolar vida com classe selecionada | Unit (Frontend) | Botão habilitado, HP calculado corretamente |
| TP-02 | Exibir lista de classes em PT-BR | Unit (Frontend) | Nomes exibidos em PT-BR |
| TP-03 | Adicionar duas notas sequenciais | Integration | Ambas notas persistidas e retornadas em lista |
| TP-04 | Arrastar item na ficha | E2E (Playwright) | Elemento acompanha o cursor sem lag |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|--------------------|
| TE-01 | Rolar vida sem classe (frontend bypass via API) | Integration | Backend retorna 422 Unprocessable Entity |
| TE-02 | Adicionar nota com título vazio | Unit (Frontend + Backend) | Validação rejeita com mensagem de erro |
| TE-03 | Nota com corpo acima do limite de 10.000 chars | Unit (Backend) | Jakarta Validation retorna 400 |
| TE-04 | Drag & drop em dispositivo touch | E2E (Playwright) | Comportamento equivalente ao mouse |

### Test Commands
- **Run frontend tests:** `cd dungeoneer-frontend && npm run test:unit`
- **Run backend tests:** `cd player-character-service/backend && mvn test`
- **Coverage check:** `cd player-character-service/backend && mvn verify` (JaCoCo ≥ 80%)
- **E2E tests:** `cd dungeoneer-frontend && npx playwright test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Migração de `notes: string` para tabela pode perder dados existentes | Med | High | Criar migration Flyway que copia texto existente para primeira nota com título "Nota importada" |
| Tradução SRD incompleta (itens sem mapeamento PT-BR) | Med | Med | Implementar fallback para o nome original em inglês com indicador visual "(sem tradução)" |
| Adoção de @dnd-kit pode introduzir regressões em outros drag & drops | Low | Med | Testes E2E cobrindo todos os pontos de drag & drop antes do merge |
| Backend pode não ter campo `name_pt_br` no banco, exigindo migration | Med | Med | Verificar schema atual antes de implementar; avaliar se tradução é só frontend |

## Specification Changes & Delta Stubs

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Adição da tabela `character_notes`, índice em `character_id`, deprecação do campo `notes TEXT` em `player_character` e migration Flyway `V5`.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Novos componentes `NoteCard`, `AddNoteForm`, mapa de traduções SRD PT-BR em `srd-translations.ts`, correção do delay no drag & drop em `DraggableScoreChip`, e guard de rolagem de vida no wizard.

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Entidade de domínio `CharacterNote`, portas de entrada `AddCharacterNoteUseCase` e `ListCharacterNotesUseCase`, porta de saída `CharacterNoteRepositoryPort`, endpoints `POST /api/characters/{id}/notes` e `GET /api/characters/{id}/notes`, e validação estrita em `RollHitPointsUseCase` para arquétipo não-nulo.

## References

- [Player Character Data Model](../../specs/player-character-data-model.md)
- [Frontend Web App Spec](../../specs/frontend-web-app.md)
- [Player Character Domain](../../specs/player-character-domain.md)
