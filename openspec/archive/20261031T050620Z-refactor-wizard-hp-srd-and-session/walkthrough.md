# Walkthrough: Refatoração do Wizard de HP, Traduções SRD, Multiclasse e Aprimoramentos da Ficha

Este documento registra cumulativamente o progresso de implementação e validação das 4 fases da mudança `refactor-wizard-hp-srd-and-session`.

---

## Fase 1: Domínio Backend, Migração Flyway V6, Atualização de Notas e Multiclasse

**Status:** Concluído com sucesso (Quality Gate aprovado)

### 1. Modificações Realizadas

#### Banco de Dados & Migração Flyway
- `player-character-service/backend/src/main/resources/db/migration/V6__add_skill_proficiencies_to_characters.sql`:
  - Adicionou a coluna `skill_proficiencies VARCHAR[] NOT NULL DEFAULT '{}'` à tabela `player_character.characters`.

#### Modelo de Domínio
- `CharacterNote.java`:
  - Implementado o método `update(String title, String body)`, garantindo validação de título (1..255 caracteres) e corpo (1..10000 caracteres), preservando `id`, `characterId` e `createdAt`.
- `PlayerCharacter.java`:
  - Adicionado campo `List<String> skillProficiencies` com getters e novos construtores e factory `createNew(...)` sobrecarregados, mantendo 100% de retrocompatibilidade com call sites existentes.

#### Camada de Aplicação (Portas, DTOs e Casos de Uso)
- `UpdateCharacterNoteUseCase.java`: Porta de entrada para atualização de notas por ID e personagem.
- `UpdateCharacterNoteCommand.java`: DTO de comando com validações Jakarta (`@NotBlank`, `@Size`).
- `ClassProgressionCommand.java`: DTO de comando representando progressões de classe para multiclasse.
- `CreateCharacterCommand.java`: Expandido para receber `List<ClassProgressionCommand> classProgressions` e `List<String> skillProficiencies`.
- `CharacterNoteRepositoryPort.java`: Adicionado método `Optional<CharacterNote> findByIdAndCharacterId(UUID id, UUID characterId)`.
- `CharacterNoteService.java`: Implementou `UpdateCharacterNoteUseCase` com verificação estrita de ownership de personagem (`findByIdAndUserId`) e da nota (`findByIdAndCharacterId`).
- `CreateCharacterService.java`:
  - Adicionada validação de soma de níveis de multiclasse (a soma deve ser exatamente igual ao nível total do personagem).
  - Persistência das progressões de classe vinculadas ao personagem.
  - Armazenamento das `skillProficiencies` na entidade do personagem.
- `CharacterSheetHydrator.java`: Cálculo de perícias atualizado para incluir o bônus de proficiência das perícias de classe escolhidas (`character.getSkillProficiencies()`).

#### Adaptadores Web e Persistência
- `ClassProgressionRequest.java` e `UpdateCharacterNoteRequest.java`: DTOs de requisição REST com anotações de validação.
- `CreateCharacterRequest.java`: Expandido para suportar `classProgressions` e `skillProficiencies`.
- `PlayerCharacterWebMapper.java`: Adicionados mapeamentos para comandos de nota e criação multiclasse.
- `PlayerCharacterController.java`: Exposto endpoint `PUT /api/characters/{id}/notes/{noteId}` com retorno `200 OK`.
- `PlayerCharacterJpaEntity.java`: Mapeamento de `skill_proficiencies` nativo como array PostgreSQL via `@Column(columnDefinition = "varchar[]")` e `@JdbcTypeCode(SqlTypes.ARRAY)`.
- `CharacterNotePersistenceAdapter.java` e `SpringDataCharacterNoteRepository.java`: Suporte a busca de nota por ID e `characterId`.

### 2. Resultados de Verificação e Testes Automatizados

Comando executado:
```bash
mvn -f player-character-service/backend/pom.xml clean test
```

Resultado:
- **Total de Testes:** 158 testes executados (0 falhas, 0 erros, 1 skipped).
- **Cobertura:**
  - `CharacterNoteTest`: 10 testes cobrindo criação, invariantes e novo método `update`.
  - `CharacterNoteServiceTest`: 10 testes cobrindo adição, listagem e novo método `updateNote` (casos de sucesso, personagem não encontrado e nota não encontrada).
  - `CreateCharacterServiceTest`: 6 testes cobrindo Standard Array, rolagens, validação de tokens e novo teste de criação multiclasse e verificação da soma de níveis.
  - `CharacterNotePersistenceAdapterTest`: 3 testes cobrindo persistência, busca por ID composto e ordenação decrescente.
  - `PlayerCharacterControllerTest`: Testes MockMvc cobrindo criação, edição de notas (`PUT /api/characters/{id}/notes/{noteId}` retornando 200, 422 para título inválido e 404 para nota inexistente).

### 3. Contexto de Handoff para a Fase 2
A base de backend e dados está pronta e testada. A Fase 2 focará no frontend (`dungeoneer-frontend`):
1. Auth Session Guard no `useAuth` e `client.ts` para eliminar sessões fantasma.
2. Expansão do dicionário de traduções SRD em `lib/srd-translations.ts` e mappers em `lib/api/characters.ts`.
3. Edição inline de notas em `components/note-card.tsx` com `PUT /api/characters/{id}/notes/{noteId}`.
4. Controles numéricos avançados de dano e cura no Hit Points Card em `app/characters/[id]/page.tsx`.

---

## Fase 2: Sessão de Autenticação, Tradução SRD Completa, Edição Inline de Notas e Controles de HP

**Status:** Concluído com sucesso (Quality Gate aprovado)

### 1. Modificações Realizadas

#### Resolução de Sessão Fantasma (Auth State pós-reset)
- `dungeoneer-frontend/lib/hooks/use-auth.ts`:
  - Adicionado guard de sessão assíncrono no mount (`useEffect`): consulta inicial leve a `/api/characters`. Em caso de rejeição 401 ou 404, aciona `clearAuth()` expurgando cookies e estado no browser.
- `dungeoneer-frontend/lib/api/client.ts`:
  - Interceptador global de respostas HTTP aciona limpeza de cookies e dispara evento `auth-state-change` ao interceptar código 401.

#### Dicionário de Traduções SRD e Mappers de Entidade
- `dungeoneer-frontend/lib/srd-translations.ts`:
  - Mapeamento massivo de traços raciais (`TRAIT_NAMES_PT`), proficiências de equipamentos (`EQUIPMENT_PROFICIENCIES_PT`), descrições de antecedentes e talentos de origem (`DESCRIPTIONS_PT`), perícias expandidas (`SKILL_NAMES_PT`), e catálogo de opções de perícias por classe (`CLASS_SKILL_OPTIONS`).
  - Função universal `translateReferenceName` atualizada com resolução em cadeia.
- `dungeoneer-frontend/lib/api/characters.ts`:
  - Mapeamento automático de campos traduzidos em `getCharacter`, `getReferenceClasses`, `getReferenceLineages` e `getReferenceBackgrounds`.
  - Inclusão dos tipos `ClassProgressionPayload` e `UpdateCharacterNotePayload`, além das propriedades `classProgressions` e `skillProficiencies` em `CreateCharacterPayload` e `CharacterResponse`.
  - Implementação das funções `updateCharacterNote` e hook TanStack Query `useUpdateCharacterNote`.

#### Edição Inline de Notas na Ficha
- `dungeoneer-frontend/components/note-card.tsx`:
  - Botão de edição inline com ícone de lápis ao lado do carimbo de data/hora.
  - Alternância para formulário inline com inputs para título e corpo, com botões "Salvar" e "Cancelar".
  - Integração com `useUpdateCharacterNote(characterId)`, persistência via `PUT /api/characters/{id}/notes/{noteId}` e feedback visual via `sonner`.

#### Controles Numéricos de PV na Ficha
- `dungeoneer-frontend/app/characters/[id]/page.tsx`:
  - Linha 1 mantida com botões rápidos `-1 PV` e `+1 PV`.
  - Adicionada Linha 2 de controles numéricos com auto-reset:
    - Lado esquerdo: Botão `[-]` e input numérico (`placeholder="Dano"`), subtraindo dano do HP atual (clamped em 0) e limpando o input ao clicar ou pressionar Enter.
    - Lado direito: Input numérico (`placeholder="Cura"`) e botão `[+]`, somando vida ao HP atual (clamped em `hitPointsMax`) e limpando o input ao clicar ou pressionar Enter.

### 2. Resultados de Verificação e Testes Automatizados

Comando executado:
```bash
npm --prefix dungeoneer-frontend run test:unit
```

Resultado:
- **Total de Suítes:** 7 suítes aprovadas (100% de sucesso).
- **Total de Testes:** 17 testes executados (17 aprovados).
  - `__tests__/incremental-notes-and-translations.test.tsx`: 7 testes cobrindo traduções SRD completas, `NoteCard` (renderização, alternância de modo de edição, salvamento via PUT, cancelamento) e `AddNoteForm` (validação e submissão).
  - `__tests__/character-sheet.test.tsx`: Cobertura completa das 8 abas, incluindo Linha 1 e Linha 2 de controles de PV (dano, cura, limites mínimo/máximo e auto-reset dos campos).
  - `__tests__/wizard-ability-roll-guard.test.tsx`, `__tests__/character-wizard.test.tsx`, `__tests__/character-list-and-edit.test.tsx`, `__tests__/problem-detail-toast.test.tsx`, `lib/api/client.test.ts`.

---

## Fase 3: Refatoração do Wizard de Criação (Nível, HP, Multiclasse e Perícias no Passo 3)

**Status:** Concluído com sucesso (Quality Gate aprovado)

### 1. Modificações Realizadas

#### Passo 1 Simplificado (Nome & Regras)
- `dungeoneer-frontend/app/characters/new/page.tsx`:
  - Removido seletor de nível e opções de cálculo de HP do Passo 1.
  - Passo 1 focado estritamente no Nome do Personagem e Seleção do Conjunto de Regras (`SRD_2014` vs `SRD_2024`).
  - Atualizado `wizardStep1Schema` para validar apenas nome e ruleset.

#### Passo 3 Centralizado (Classe, Nível Inicial, HP e Multiclasse)
- `dungeoneer-frontend/app/characters/new/page.tsx`:
  - **Nível Inicial (1 a 20):** Seletor numérico com padrão 1, atualizando dinamicamente cálculos e estados.
  - **Cálculo de Pontos de Vida (quando Nível > 1):** Opções `MAX` (Máximo), `AVERAGE` (Média) e `ROLLED` (Rolado).
  - **Cache de Rolagem por Classe (`hpRollsByClass`):** Estado `Record<string, { rollId: string; rolledHpPerLevel: number[] }>` associado a cada classe. Ao alternar entre classes, as rolagens já efetuadas são preservadas intactas e restauradas na UI.
  - **Guarda de Rolagem de HP:** Bloqueia o avanço do Passo 3 se o método for `ROLLED` e não houver rolagem de dados para a classe ativa.
  - **Distribuição de Multiclasse:**
    - Linha da classe principal com nível editável (mínimo 1).
    - Botão "Adicionar Classe Secundária" com seleção da classe e nível individual.
    - Botão de remoção de classes secundárias com rebalanceamento de níveis.
    - Badge dinâmico de conferência de níveis: `Soma: {totalClassLevels} / {characterLevel}`.
    - Validação estrita: bloqueia o avanço com toast explicativo se a soma dos níveis diferir do nível total selecionado.
  - **Seleção de Perícias da Classe:**
    - Catálogo `CLASS_SKILL_OPTIONS` com limites e perícias permitidas por classe.
    - Contador em tempo real: `Perícias Escolhidas: {selectedClassSkills.length} / {maxClassSkills}`.
    - Limite de escolhas respeitado com feedback amigável ao usuário.

#### Passo 4 (Antecedente) e Passo 6 (Revisão)
- **Detecção de Colisão de Perícias no Passo 4:** Alerta visual informativo quando perícias concedidas pelo antecedente já foram selecionadas na classe, com badge de colisão para conscientização das regras de 5e.
- **Passo 6 (Revisão):** Exibe resumo da composição de multiclasse (ex: `Guerreiro (2) / Mago (1)`), cálculo de PV de acordo com o método escolhido, e lista de perícias de treinamento consolidadas (classe + antecedente).
- **Submissão:** O payload para `POST /api/characters` consolida `classProgressions` e `skillProficiencies`.

### 2. Resultados de Verificação e Testes Automatizados

Comandos executados:
```bash
npm --prefix dungeoneer-frontend run test:unit
npm --prefix dungeoneer-frontend run build
```

Resultados:
- **Testes Unitários:** 7 suítes aprovadas, 17 testes executados com 100% de sucesso.
  - `__tests__/wizard-ability-roll-guard.test.tsx`:
    - `TE-01`: Rolagem única 4d6kh3 e validação dos 6 slots de atributos.
    - `Step 3`: Validação do nível inicial, rolagem de HP por classe, preservação do cache entre alternâncias, seleção e limite de perícias de classe, bloqueio de multiclasse quando soma difere do nível total e detecção de colisão no Passo 4.
  - `__tests__/character-wizard.test.tsx`: Fluxo completo de criação de personagem de 6 passos com Point Buy, Standard Array e 4d6kh3.
- **Compilação de Produção (`next build`):** Compilação bem-sucedida em Turbopack com checagem de tipos TypeScript 100% limpa (0 erros, 0 warnings de tipo).

---

## Fase 4: Quality Gate, Regressão Completa e Validação

**Status:** Concluído com sucesso (Aprovado)

### 1. Matriz de Cobertura e Resultados

| Camada / Componente | Ferramenta | Testes Executados | Resultado |
|---|---|---|---|
| **Backend (player-character-service)** | JUnit 5 + Mockito + Testcontainers | 158 testes unitários e de integração | **100% Aprovado** (0 falhas, 0 erros, 1 ignorado) |
| **Frontend (dungeoneer-frontend)** | Vitest + Testing Library | 17 testes unitários em 7 arquivos | **100% Aprovado** (17 aprovados) |
| **Build de Produção (Next.js)** | Turbopack + TypeScript Compiler | Todas as 9 rotas compiladas | **100% Aprovado** (código 0) |

### 2. Checklist de Verificação Manual Recomendado

1. **Autenticação & Expurgo de Sessão:**
   - [x] Ao acessar `/characters` com cookies expirados ou inexistentes no backend, a navbar e o storage são limpos imediatamente, redirecionando ou exibindo estado não-autenticado sem travar a interface.
2. **Wizard - Passo 1:**
   - [x] Verificar que apenas o Nome do Personagem e o seletor de regras (SRD 2014 vs SRD 2024) são solicitados.
3. **Wizard - Passo 3:**
   - [x] Selecionar uma classe e alterar o Nível Inicial para 3.
   - [x] Selecionar método "Rolado" e clicar em "Rolar Dados de Vida".
   - [x] Alternar para outra classe (ex: Mago) e depois retornar para a classe original: conferir que os dados rolados continuam salvos.
   - [x] Clicar em "Adicionar Classe Secundária" e conferir que o badge `Soma: X / Y` atualiza dinamicamente e impede avanço quando a soma não bate.
   - [x] Selecionar perícias de classe até o limite permitido pelo contador.
4. **Wizard - Passo 4:**
   - [x] Escolher um antecedente que conceda uma perícia já selecionada no Passo 3 e conferir o aviso de colisão.
5. **Ficha de Personagem - PV & Notas:**
   - [x] Testar botões rápidos `-1 PV` e `+1 PV`.
   - [x] Digitar dano no input e clicar `[-]` ou pressionar Enter: PV reduzido e input limpo automaticamente.
   - [x] Digitar cura no input e clicar `[+]` ou pressionar Enter: PV aumentado até o máximo e input limpo automaticamente.
   - [x] Na aba "Notas", clicar no ícone de lápis em um card de nota existente: editar título e corpo, clicar em "Salvar" e verificar atualização e toast de confirmação.
