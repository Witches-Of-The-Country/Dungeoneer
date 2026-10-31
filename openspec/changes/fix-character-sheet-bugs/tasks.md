# Tasks: Fix Character Sheet Bugs

> **Change ID:** `fix-character-sheet-bugs`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### Bug 1 — Rolagem de Vida Após Seleção de Classe
- [ ] Adicionar guard no componente de rolagem de vida (wizard) para desabilitar botão quando `class` é nulo
- [ ] Exibir tooltip/mensagem "Selecione uma classe primeiro" quando botão estiver desabilitado
- [x] Adicionar validação no backend (use case) que rejeita roll de HP sem classe definida

### Bug 2 — Tradução SRD para PT-BR
- [ ] Verificar schema do banco: checar se tabelas SRD possuem coluna `name_pt_br`
- [ ] Criar arquivo `srd-translations.ts` com mapa de slugs → nomes PT-BR para classes
- [ ] Criar mapa PT-BR para raças/linhagens
- [ ] Criar mapa PT-BR para backgrounds
- [ ] Criar mapa PT-BR para magias (spells)
- [ ] Criar mapa PT-BR para habilidades e proficiências
- [ ] Aplicar traduções nos componentes de listagem (TanStack Query selectors)
- [ ] Implementar fallback para nome original quando tradução não existe

### Bug 3 — Notas Incrementais
- [x] Criar migration Flyway: tabela `character_note` com `id, character_id, title, body, created_at`
- [x] Criar migration Flyway: migrar dados existentes de `notes TEXT` para primeira `character_note`
- [x] Criar entidade de domínio `CharacterNote` (sem imports de framework)
- [x] Criar DTO `CharacterNoteRequest` com validações Jakarta (`@NotBlank`, `@Size`)
- [x] Criar port de saída `CharacterNoteRepository`
- [x] Criar use case `AddCharacterNoteUseCase`
- [x] Criar use case `ListCharacterNotesUseCase`
- [x] Criar JPA entity `CharacterNoteJpaEntity` e `CharacterNoteJpaRepository`
- [x] Criar endpoint `POST /characters/{id}/notes`
- [x] Criar endpoint `GET /characters/{id}/notes`
- [ ] Criar componente frontend `NoteCard` (exibe título, corpo, data)
- [ ] Criar componente frontend `AddNoteForm` (campos título e corpo)
- [ ] Integrar `AddNoteForm` e lista de `NoteCard` na seção de notas da ficha
- [ ] Substituir campo `notes: string` por `notes: CharacterNote[]` no tipo frontend

### Bug 4 — Lag no Drag & Drop
- [ ] Identificar a biblioteca/implementação atual de drag & drop no projeto
- [ ] Avaliar solução: migrar para `@dnd-kit/core` ou corrigir implementação atual
- [ ] Implementar correção (pointer events + transform CSS sem re-render)
- [ ] Validar correção em movimentos rápidos do mouse

## Test Tasks

- [ ] Escrever teste unitário (Frontend): botão de rolar vida desabilitado sem classe
- [ ] Escrever teste unitário (Frontend): botão habilitado após seleção de classe
- [x] Escrever teste de integração (Backend): rejeitar HP roll sem classe via API
- [ ] Escrever testes unitários (Frontend): validar renderização de nomes PT-BR nas listas
- [ ] Escrever teste unitário: `AddNoteForm` valida campos obrigatórios
- [x] Escrever teste de integração: `POST /characters/{id}/notes` persiste nota
- [x] Escrever teste de integração: `GET /characters/{id}/notes` retorna lista ordenada
- [x] Escrever teste unitário: `CharacterNoteRequest` rejeita título vazio
- [x] Escrever teste unitário: `CharacterNoteRequest` rejeita corpo acima de 10.000 chars
- [ ] Escrever teste E2E (Playwright): drag & drop sem lag perceptível
- [ ] Escrever teste E2E (Playwright): adicionar duas notas e verificar que ambas aparecem

## Documentation Tasks

- [ ] Atualizar `openspec/specs/player-character-data-model.md` via delta stub (tabela `character_notes` e Flyway V5)
- [ ] Atualizar `openspec/specs/frontend-web-app.md` via delta stub (novos componentes, traduções e DnD fix)
- [ ] Atualizar `openspec/specs/player-character-domain.md` via delta stub (use cases e domínio de notas)
- [ ] Atualizar OpenAPI spec com novos endpoints de notas

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] JaCoCo coverage ≥ 80% no backend
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
