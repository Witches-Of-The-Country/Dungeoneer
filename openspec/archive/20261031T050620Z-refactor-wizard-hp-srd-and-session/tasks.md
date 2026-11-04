# Tasks: Refactor Wizard HP, Full SRD Translations, Multiclassing and Character Sheet Enhancements

> **Change ID:** `refactor-wizard-hp-srd-and-session`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** complete

All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

### 1. Wizard UX: Nível Inicial, Rolagem de HP e Multiclasse no Passo 3
- [x] Mover seletor de nível inicial do Passo 1 para o Passo 3 (Classe) em `app/characters/new/page.tsx`
- [x] Mover opções de cálculo de HP (`MAX`, `AVERAGE`, `ROLLED`) e botão de rolagem de dados para o Passo 3
- [x] Simplificar Passo 1 mantendo apenas Nome do Personagem e Seleção do Conjunto de Regras (`SRD_2014` vs `SRD_2024`)
- [x] Implementar cache de rolagens por classe `hpRollsByClass: Record<string, HpRollData>` no estado do wizard
- [x] Implementar alocação de Multiclasse no Passo 3 para Nível > 1: Classe principal (nível 1 fixo) + botão de adicionar classes secundárias distribuindo níveis restantes
- [x] Validar que a soma dos níveis das classes é exatamente igual ao nível total selecionado antes de avançar do Passo 3

### 2. Escolha de Perícias de Classe no Wizard
- [x] Criar catálogo de opções e limites de perícias por classe em `lib/srd-translations.ts` ou `lib/api/characters.ts`
- [x] No Passo 3, exibir seleção de perícias permitidas pela classe principal com contador de escolhas disponíveis
- [x] No Passo 4 (Antecedente), exibir aviso/badge se uma perícia da classe colidir com o antecedente, permitindo trocar
- [x] Criar migration Flyway V6 adicionando `skill_proficiencies VARCHAR[]` na tabela `player_character.characters`
- [x] Atualizar DTOs `CreateCharacterRequest`, `CreateCharacterCommand` e entidade de domínio para salvar perícias consolidadas

### 3. Edição Inline de Notas na Ficha de Personagem
- [x] Criar port de entrada `UpdateCharacterNoteUseCase` e DTO `UpdateCharacterNoteCommand` no backend
- [x] Implementar método `updateNote` no `CharacterNoteService` com verificação estrita de ownership
- [x] Criar endpoint `PUT /api/characters/{id}/notes/{noteId}` no `PlayerCharacterController`
- [x] Adicionar mutação `useUpdateCharacterNote` e função de API no frontend `lib/api/characters.ts`
- [x] Adicionar modo de edição inline no `NoteCard` com formulário editável de título, corpo e botões Salvar/Cancelar

### 4. Controles Numéricos de Ajuste de PV na Ficha
- [x] No Card de PV de `app/characters/[id]/page.tsx`, adicionar linha 2 de controles:
  - À esquerda: Botão `-` seguido de input numérico de dano
  - À direita: Input numérico de cura seguido de botão `+`
- [x] Implementar handlers para aplicar dano e cura (respeitando limites 0 e `hitPointsMax`) e limpar o input após o clique

### 5. Tradução Completa de Textos e Conteúdo SRD para PT-BR
- [x] Adicionar mapa de traduções para traços raciais (`traits_summary`) em `lib/srd-translations.ts`
- [x] Adicionar mapa de traduções para descrições de antecedentes e talentos de origem (`originFeatDescription`)
- [x] Adicionar mapa de traduções para proficiências de equipamentos (armaduras, armas e ferramentas)
- [x] Atualizar seletores de consulta em `lib/api/characters.ts` para aplicar mappers em todas as entidades SRD

### 6. Resolução da Sessão Fantasma (Auth State pós-reset)
- [x] No `useAuth`, adicionar verificação assíncrona de sessão no mount: acionar `clearAuth()` se o backend rejeitar com 401/404
- [x] No cliente HTTP `lib/api/client.ts`, acionar `clearAuth()` em qualquer interceptação 401

## Test Tasks

- [x] Testes unitários do Wizard: validar nível, HP e seleção de perícias no Passo 3
- [x] Teste unitário de Multiclasse: distribuição de níveis e soma obrigatória igual ao nível total
- [x] Teste de persistência de HP por classe ao alternar e retornar a uma classe
- [x] Teste de edição inline de notas no backend (`PUT /api/characters/{id}/notes/{noteId}`)
- [x] Teste de edição inline de notas no frontend (`NoteCard`)
- [x] Teste dos controles numéricos de dano e cura com reset do input
- [x] Testes unitários de traduções SRD expandidas em `__tests__/incremental-notes-and-translations.test.tsx`
- [x] Teste de integridade de sessão no `useAuth` limpando storage em rejeições 401/404

## Documentation Tasks

- [x] Atualizar `specs/frontend-web-app-delta.md`
- [x] Criar delta spec `specs/player-character-domain-delta.md`
- [x] Criar delta spec `specs/player-character-data-model-delta.md`
- [x] Elaborar walkthrough com cobertura dos novos recursos (Fase 1 registrada)

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
