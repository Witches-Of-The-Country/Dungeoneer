# Proposal: Refactor Wizard HP, Full SRD Translations, Multiclassing and Character Sheet Enhancements

> **Change ID:** `refactor-wizard-hp-srd-and-session`
> **Status:** draft
> **Author:** @aluno
> **Created:** 2026-10-30
> **Last Updated:** 2026-10-30

---

## Summary

Este change consolida um conjunto de aprimoramentos essenciais na criação e gestão de personagens do Dungeoneer:
1. Reorganiza a arquitetura do Wizard movendo a escolha do nível inicial e cálculo/rolagem de HP para o Passo 3 (Classe).
2. Adiciona suporte a persistência de rolagens de HP por classe (`Record<classId, HpRollData>`), permitindo alternar classes sem perda de dados.
3. Introduz a seleção de perícias de classe no Passo 3, com resolução de sobreposições em relação ao Antecedente.
4. Permite a distribuição de níveis entre múltiplas classes (Multiclasse) para personagens que iniciam em níveis acima do 1 (Classe Principal no nível 1 + alocação dos níveis restantes).
5. Expande o dicionário de traduções SRD em PT-BR para cobrir exaustivamente traços raciais (*traits_summary*), descrições de antecedentes, talentos de origem e proficiências de equipamentos.
6. Torna as notas do personagem editáveis de forma inline no próprio `NoteCard` com suporte a `PUT /api/characters/{id}/notes/{noteId}`.
7. Adiciona controles numéricos avançados de PV na ficha de personagem (botão `-` seguido de input numérico para dano, e input numérico seguido de botão `+` para cura com limpeza automática após o clique).
8. Elimina sessões fantasmas no frontend pós-reset de volumes validando a autenticidade do token no startup contra o backend.

## Motivation

- **Ergonomia e Coerência Conceitual no Wizard:** O dado de vida (`hit_die`), perícias e proficiências dependem da classe. Configurar o nível no Passo 1 antes de selecionar a classe gerava incoerência e guards artificiais. No Passo 3, todo o contexto da classe está disponível.
- **Multiclasse em Níveis Acima do 1:** Em D&D 5e/2024, personagens criados em níveis superiores ao 1 podem optar por multiclasse. A classe de 1º nível dita as perícias iniciais, resistências e dado de vida máximo inicial, enquanto os níveis subsequentes podem ser livremente distribuídos entre as classes disponíveis.
- **Escolha de Perícias de Classe:** Apenas as perícias do antecedente vinham pré-definidas. O jogador não podia escolher as perícias de sua classe (ex: 2 para Guerreiro, 4 para Ladino, 3 para Bardo), elemento fundamental da ficha.
- **Edição de Notas:** As notas eram criadas de forma incremental, mas não podiam ser retificadas ou atualizadas caso o jogador quisesse corrigir um erro ou complementar anotações de sessão.
- **Ajustes Rápidos de Dano e Cura:** Em combate, personagens sofrem danos e curas de valores arbitrários (ex: 8 de dano de uma magia ou 15 de cura de uma poção). Clicar repetidamente em `-1 PV` ou `+1 PV` é impraticável. Controles com inputs numéricos dedicados são indispensáveis.
- **Completude das Traduções SRD:** Traços como *Darkvision*, *Fey Ancestry*, proficiências de equipamentos e descrições de antecedentes precisam estar em português em toda a interface.
- **Sessão Fantasma pós-reset:** Após `docker-compose down -v`, cookies e `localStorage` no navegador mantinham usuários inexistentes logados no frontend sem validação prévia.

## Scope

### In Scope
- Mover nível inicial e cálculo/rolagem de HP do Passo 1 para o Passo 3 do Wizard.
- Implementar cache de rolagens de HP por classe (`hpRollsByClass`) no estado do Wizard.
- Implementar seleção de perícias de classe no Passo 3 do Wizard com validação de limites por classe e detecção de sobreposição com o Antecedente no Passo 4.
- Implementar interface e suporte de dados para Multiclasse na criação de personagem (nível > 1: Classe principal no nível 1 + alocação dos níveis restantes).
- Criar endpoint `PUT /api/characters/{id}/notes/{noteId}` no backend e modo de edição inline no `NoteCard` no frontend.
- Implementar controles numéricos de PV na ficha: linha 1 com `-1 PV` e `+1 PV`; linha 2 com `[-] [input]` (dano) e `[input] [+]` (cura) com reset para vazio após aplicação.
- Expandir `lib/srd-translations.ts` com traços raciais, descrições de antecedentes, talentos de origem e proficiências.
- Implementar validação de sessão ativa no startup do `useAuth` e interceptação 401 no `client.ts` chamando `clearAuth()`.
- Atualizar suites de testes automatizados unitários e de integração no frontend e backend.

### Out of Scope
- Árvores complexas de talentos épicos ou subclasses de homebrew não homologadas pelo SRD.
- Exclusão em lote de notas (apenas edição inline e criação incremental).

## Proposed Solution

### Approach

1. **Wizard: Nível, HP e Multiclasse no Passo 3 (`app/characters/new/page.tsx`):**
   - O Passo 1 mantém apenas Nome e Conjunto de Regras (`SRD_2014` / `SRD_2024`).
   - No Passo 3, o usuário define o Nível Inicial Total do personagem (1 a 20).
   - O usuário escolhe a **Classe Principal (Nível 1)**.
   - Se `Nível Total > 1`, exibe-se um painel de "Distribuição de Níveis (Multiclasse)":
     - Nível 1 atribuído automaticamente à Classe Principal.
     - Botão "Adicionar Classe Secundária" permitindo selecionar outras classes e alocar níveis até que a soma atinja o Nível Total.
   - Cálculo de HP: Para a classe principal e classes adicionais, exibe o método (`MAX`, `AVERAGE`, `ROLLED`) e gerencia `hpRollsByClass: Record<string, HpRollResponse>`.

2. **Wizard: Seleção de Perícias da Classe Principal (`app/characters/new/page.tsx`):**
   - Baseado na Classe Principal, exibe-se a lista de perícias elegíveis e o limite de escolhas (ex: Guerreiro escolhe 2, Ladino escolhe 4, Bardo escolhe 3 quaisquer).
   - O estado do wizard registra `selectedClassSkills: string[]`.
   - No Passo 4 (Antecedente), se o antecedente fornecer uma perícia que o jogador já havia selecionado na classe, o sistema avisa com um badge e permite trocar a escolha da classe.
   - O backend salva a lista consolidada em `player_character.characters.skill_proficiencies` (migration Flyway V6).

3. **Edição Inline de Notas (`components/note-card.tsx` e `lib/api/characters.ts`):**
   - O backend adiciona a porta de entrada `UpdateCharacterNoteUseCase`, o endpoint `PUT /api/characters/{id}/notes/{noteId}` e a validação de ownership.
   - O `NoteCard` ganha um botão de ação com ícone de lápis. Ao clicar, o card alterna para campos de formulário inline (título e corpo) com botões "Salvar" e "Cancelar".
   - Ao salvar, dispara `useUpdateCharacterNote` atualizando a nota instantaneamente no React Query.

4. **Controles Numéricos de Ajuste de PV na Ficha (`app/characters/[id]/page.tsx`):**
   - No Card de Pontos de Vida do Resumo:
     - Linha 1: Botões `-1 PV` e `+1 PV`.
     - Linha 2 (Ajustes de Montante):
       - À esquerda (Dano): Botão `-` seguido de um input numérico (`placeholder="Dano"`). Clicar no `-` reduz os PV atuais pelo valor digitado e limpa o input.
       - À direita (Cura): Input numérico (`placeholder="Cura"`) seguido do botão `+`. Clicar no `+` adiciona o valor aos PV atuais (limitado ao PV máximo) e limpa o input.

5. **Dicionário SRD Expandido (`lib/srd-translations.ts` e `lib/api/characters.ts`):**
   - Mapeamentos exaustivos para traços raciais (*Darkvision* -> *Visão no Escuro*, *Fey Ancestry* -> *Ancestralidade Feérica*, etc.), proficiências de armas (*Simple*, *Martial*), armaduras (*Light*, *Medium*, *Heavy*, *Shields*) e descrições de antecedentes.
   - Mappers aplicados em todas as consultas TanStack Query.

6. **Validação de Sessão no Frontend (`lib/hooks/use-auth.ts` e `lib/api/client.ts`):**
   - `useAuth` valida a integridade do token armazenado no mount. Se a chamada ao backend falhar com 401 ou usuário não encontrado (após limpeza de volumes), executa `clearAuth()`.
   - `apiFetch` limpa credenciais salvas em qualquer resposta 401.

### Impacted Files

| File | Change Type | Notes |
|------|-------------|-------|
| `dungeoneer-frontend/app/characters/new/page.tsx` | Modify | Nível, HP, Multiclasse e Perícias de classe no Passo 3 |
| `dungeoneer-frontend/components/note-card.tsx` | Modify | Adicionar modo de edição inline com botões Salvar/Cancelar |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Modify | Controles numéricos de dano/cura com reset automático |
| `dungeoneer-frontend/lib/srd-translations.ts` | Modify | Expandir dicionários de traços, proficiências e descrições |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | Tipos de multiclasse, perícias, `updateCharacterNote`, mappers SRD |
| `dungeoneer-frontend/lib/api/auth.ts` | Modify | Helper de validação de sessão e `clearAuth` sincronizado |
| `dungeoneer-frontend/lib/hooks/use-auth.ts` | Modify | Validação de sessão no startup em `useEffect` |
| `dungeoneer-frontend/lib/api/client.ts` | Modify | Expurgar cookies e storage em interceptações de 401 |
| `player-character-service/backend/src/main/resources/db/migration/V6__add_skill_proficiencies_to_characters.sql` | Create | Coluna `skill_proficiencies VARCHAR[]` em `characters` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/UpdateCharacterNoteUseCase.java` | Create | Port de entrada para edição de notas |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CharacterNoteService.java` | Modify | Implementar `updateNote` com validação de ownership |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java` | Modify | Endpoint `PUT /api/characters/{id}/notes/{noteId}` |
| `dungeoneer-frontend/__tests__/wizard-ability-roll-guard.test.tsx` | Modify | Adaptar testes de rolagem e nível para o Passo 3 |
| `dungeoneer-frontend/__tests__/character-wizard.test.tsx` | Modify | Testar fluxo completo com perícias e multiclasse |
| `dungeoneer-frontend/__tests__/character-sheet.test.tsx` | Modify | Testar edição de nota inline e controles numéricos de PV |
| `dungeoneer-frontend/__tests__/incremental-notes-and-translations.test.tsx` | Modify | Testar traduções expandidas e edição de notas |

### Data Model Changes

1. **Migration Flyway V6 (`player_character.characters`):**
```sql
ALTER TABLE player_character.characters
    ADD COLUMN IF NOT EXISTS skill_proficiencies VARCHAR[] NOT NULL DEFAULT '{}';
```

2. **Multiclasse (`player_character.class_progressions`):**
Já suporta múltiplos registros por personagem (`character_id`, `archetype_id`, `class_level`).

## Acceptance Criteria

- [ ] **Given** o Passo 3 do Wizard **When** o usuário escolhe a classe inicial **Then** exibe a lista de perícias elegíveis e permite escolher até o limite da classe
- [ ] **Given** Nível Total > 1 no Passo 3 **When** o usuário distribui níveis entre classes **Then** a soma dos níveis deve ser exatamente igual ao Nível Total para permitir avançar
- [ ] **Given** rolagem de HP realizada para a classe A **When** o usuário troca para classe B e depois retorna para A **Then** o valor rolado para A é recuperado do cache
- [ ] **Given** uma nota na lista de notas da ficha **When** o usuário clica em Editar no NoteCard **Then** campos inline aparecem, permitindo salvar alterações via PUT
- [ ] **Given** a ficha de personagem **When** o usuário insere um valor no input de dano e clica no botão `-` **Then** os PV atuais diminuem por aquele montante e o input é limpo
- [ ] **Given** a ficha de personagem **When** o usuário insere um valor no input de cura e clica no botão `+` **Then** os PV atuais aumentam (limitados ao máximo) e o input é limpo
- [ ] **Given** o wizard e a ficha **When** linhagens, traços, antecedentes e proficiências são renderizados **Then** todos aparecem em PT-BR
- [ ] **Given** cookies de usuário antigo com banco de dados limpo **When** a aplicação carrega **Then** o sistema limpa cookies e storage e exibe estado deslogado

## Defensive Security & Validation Considerations

### Input Validation
- Edição de nota valida título (`@NotBlank`, `max=255`) e corpo (`@NotBlank`, `max=10000`).
- Dano e cura numéricos validam valores inteiros positivos.
- Soma de níveis de multiclasse validada para nunca ultrapassar 20 nem divergir do nível total declarado.

### Secret Isolation
- Nenhuma credencial manipulada ou exposta.

### Authorization & Authentication Impact
- `PUT /api/characters/{id}/notes/{noteId}` valida que o personagem e a nota pertencem ao `userId` do token.
- Sessões fantasmas eliminadas impedindo impersonação em clientes com storage obsoleto.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Escolha de perícias de classe no Passo 3 | Unit (Frontend) | Perícias salvas e refletidas na ficha |
| TP-02 | Criação de personagem multiclasse (ex: Guerreiro 3 / Ladino 2) | Unit + Integration | Progressões salvas corretamente |
| TP-03 | Edição inline de nota existente | Unit + Integration | PUT persistido e exibido no card |
| TP-04 | Dano e cura com inputs numéricos na ficha | Unit (Frontend) | PV ajustado e campos limpos |
| TP-05 | Tradução de traços raciais e proficiências | Unit (Frontend) | Textos renderizados em PT-BR |

### Edge Case Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Multiclasse com soma de níveis divergente | Unit (Frontend) | Bloqueia avanço com mensagem de erro |
| TE-02 | Edição de nota de outro usuário via API | Integration (Backend) | Retorna 404/403 |
| TE-03 | Dano que excede os PV atuais | Unit (Frontend) | PV cai para 0, nunca negativo |

### Test Commands
- Frontend: `npm --prefix dungeoneer-frontend run test:unit`
- Frontend Build: `npm --prefix dungeoneer-frontend run build`
- Backend: `mvn -f player-character-service/backend/pom.xml test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Inconsistência na soma de níveis ao alternar classes em multiclasse | Med | Med | Validação reativa síncrona no frontend bloqueando avanço enquanto a soma não atingir exatamente o nível total |
| Compatibilidade de migração da coluna `skill_proficiencies` para personagens pré-existentes | Low | Med | Migration Flyway V6 define `DEFAULT '{}'` e `NOT NULL`, garantindo retrocompatibilidade total com registros legados |
| Condição de corrida na verificação assíncrona de sessão no mount do `useAuth` | Low | Low | Manter flag `isLoading` no `useAuth` e só despachar logout se o backend responder explicitamente com erro 401/404 |
| Perda de edições de notas caso o usuário feche a aba durante a edição inline | Low | Low | Salvar rascunho temporário no estado local e confirmação visual de salvamento com Sonner toast |

## Specification Changes & Delta Stubs

### `specs/frontend-web-app-delta.md`
- Atualização da máquina de estados do Wizard (Perícias no Passo 3, Multiclasse, Nível e HP), edição de notas e controles de PV.

### `specs/player-character-domain-delta.md`
- Portas e use cases de atualização de notas (`UpdateCharacterNoteUseCase`).

### `specs/player-character-data-model-delta.md`
- Adição da coluna `skill_proficiencies` em `player_character.characters` (Flyway V6).
