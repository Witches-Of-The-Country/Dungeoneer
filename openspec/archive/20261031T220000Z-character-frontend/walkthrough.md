# Walkthrough: Implementação Completa do Frontend de Personagens (`character-frontend`)

> **Change ID:** `character-frontend`
> **Módulo Alvo:** `dungeoneer-frontend`
> **Gateway Alvo:** `process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:9090'`
> **Idioma da Interface:** Português (`pt-BR`)

---

## Resumo Consolidado por Fase

### Fase 1: Dependências, Ambiente de Testes e Cliente de API
- Instaladas as bibliotecas de produção `@tanstack/react-query`, `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities` e `sonner`.
- Instaladas e configuradas as ferramentas de teste `vitest`, `@vitejs/plugin-react`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom` e `@playwright/test` (`vitest.config.ts` e `vitest.setup.ts`).
- Criado o cliente HTTP centralizado em `lib/api/client.ts` com anexação automática do cabeçalho `Authorization: Bearer <token>`, tradução de erros RFC 7807 (`ProblemDetail`) para notificações toast em `pt-BR` via `sonner`, e redirecionamento automático para `/login` em respostas `401 Unauthorized`.
- Configurados `QueryClientProvider` e `Toaster` em `components/providers.tsx` e integrados em `app/layout.tsx`.

### Fase 2: Autenticação via Gateway, Sessão na Navbar e Proteção de Rotas
- Atualizado `lib/api/auth.ts` e `lib/hooks/use-auth.ts` para rotear `POST /api/auth/login` e `POST /api/auth/register` através do API Gateway (`:9090`), persistindo token JWT e perfil de usuário em `localStorage` e cookies.
- Atualizados `components/login-form.tsx` e `components/register-form.tsx` com validação Zod + React Hook Form, exibição de toasts e redirecionamento para `/characters`.
- Atualizada `components/navbar.tsx` exibindo link "Meus Personagens", identificação do usuário autenticado e ação "Sair".
- Adicionada proteção de rotas para `/characters/**` em `middleware.ts` e `app/characters/layout.tsx`.

### Fase 3: Hooks TanStack Query e Lista de Personagens (`/characters`)
- Implementado `lib/api/characters.ts` contendo os hooks `useCharacters`, `useCharacter(id)`, `useCreateCharacter`, `useUpdateCharacter`, `useDeleteCharacter`, `useRollAbilityScores`, `useRollHitPoints`, `useReferenceClasses`, `useReferenceLineages` e `useReferenceBackgrounds`, além de utilitários de regras D&D 5e (cálculo de modificadores, bônus de proficiência, tabela de 27 pontos de Point Buy e lista das 18 perícias em `pt-BR`).
- Construída a página `app/characters/page.tsx` com grid de cards Dark Fantasy RPG (nome, selo `5e 2014` / `5e 2024`, nível, classe, linhagem, barra de PV, CA e ações rápidas `Abrir Ficha`, `Editar` e `Excluir`), estado vazio temático e modal de confirmação de exclusão (`components/delete-character-modal.tsx`).

### Fase 4: Wizard de Criação de Personagem em 6 Passos (`/characters/new`)
- Implementado `app/characters/new/page.tsx` reutilizando o componente `Stepper`:
  1. **Passo 1 — Nome & Regras**: Nome (`1–100` caracteres validado com Zod), seletor `SRD_2014` vs `SRD_2024`, nível inicial (`1–20`) e método de cálculo de PV (`MAX`, `AVERAGE`, `ROLLED` com rolagem no servidor via `POST /api/characters/hp-rolls`).
  2. **Passo 2 — Linhagem**: Consulta dinâmica de linhagens com exibição de deslocamento, porte, traços raciais e bônus de atributo fixos (`SRD_2014`).
  3. **Passo 3 — Classe**: Consulta dinâmica de classes com dado de vida, atributo principal, salvaguardas e indicador de conjuração.
  4. **Passo 4 — Antecedente**: Consulta dinâmica de antecedentes com perícias e, para `SRD_2024`, card de Talento de Origem e alocador interativo de bônus de atributo (`+2/+1` ou `+1/+1/+1`).
  5. **Passo 5 — Atributos**: Suporte completo aos 3 métodos:
     - `Rolagem (4d6kh3)`: Botão de chance única (`POST /api/characters/ability-rolls`) com bloqueio persistido em `sessionStorage` e alocação via drag-and-drop (`@dnd-kit/core`) ou clique nos 6 atributos (`FOR`, `DES`, `CON`, `INT`, `SAB`, `CAR`).
     - `Valores Padrão`: Alocação de `[15, 14, 13, 12, 10, 8]` via drag-and-drop ou clique.
     - `Compra de Pontos`: Controles `+/-` (`8..15`) com contador em tempo real de `27` pontos.
  6. **Passo 6 — Revisão & Criação**: Resumo consolidado de estatísticas derivadas (PV, CA, Iniciativa, Proficiência) e submissão `POST /api/characters` redirecionando para `/characters/[id]`.

### Fase 5: Ficha de Personagem em 8 Abas (`/characters/[id]`)
- Implementada `app/characters/[id]/page.tsx` com cabeçalho completo (incluindo badge `Escolhas de Nível Pendentes` quando `hasPendingLevelChoices` é verdadeiro) e navegação em 8 abas:
  - **Abas Interativas com Dados Reais**: `Resumo` (PV atual/máx/temp com ajuste rápido `-1 PV` / `+1 PV`, CA, Iniciativa, Deslocamento, Proficiência, Dados de Vida e Salvaguardas), `Atributos` (6 cards detalhados), `Perícias` (todas as 18 perícias em `pt-BR` com indicador de proficiência e modificador total) e `Notas` (edição e salvamento via `PUT /api/characters/{id}`).
  - **Abas Placeholder ("Em breve")**: `Ações`, `Magias`, `Inventário` e `Talentos` (exibindo também o Talento de Origem 2024 quando presente).

### Fase 6: Página de Edição de Personagem (`/characters/[id]/edit`)
- Implementada `app/characters/[id]/edit/page.tsx` com React Hook Form + Zod para edição de `name`, `hitPointsCurrent` (validado contra `hitPointsMax`), `hitPointsTemp` e `notes`, resumo somente-leitura das propriedades imutáveis de criação e botão de exclusão com modal de confirmação.

### Fase 7 & 8: Suíte de Testes Automatizados (Shift-Left) e Verificação Final
- Implementados 6 arquivos de teste (9 casos de teste no total):
  - `lib/api/client.test.ts`: Anexação de Bearer token e formatação de erros RFC 7807.
  - `__tests__/character-wizard.test.tsx` (**TP-01**): Fluxo completo de 6 passos, Point Buy, rolagem `4d6kh3`, alocação nos 6 slots e submissão `POST /api/characters`.
  - `__tests__/character-sheet.test.tsx` (**TP-02**): Renderização e troca entre as 8 abas, estatísticas de combate, 18 perícias, edição de notas e estados "Em breve".
  - `__tests__/character-list-and-edit.test.tsx` (**TP-03**): Listagem de cards, confirmação de exclusão (`DELETE /api/characters/{id}`) e formulário de edição (`PUT /api/characters/{id}`).
  - `__tests__/wizard-ability-roll-guard.test.tsx` (**TE-01**): Bloqueio de segunda rolagem `4d6kh3` e bloqueio do botão "Próximo" enquanto houver slot de atributo vazio.
  - `__tests__/problem-detail-toast.test.tsx` (**TE-02**): Exibição de toasts em `pt-BR` para erros `409 Conflict` e `422 Unprocessable Entity` (`ProblemDetail`).

---

## Resultados de Verificação (Quality Gates)

```text
✓ npm --prefix dungeoneer-frontend run lint       -> 0 errors
✓ npm --prefix dungeoneer-frontend run build      -> Compiled successfully (Next.js 16.2.6 Turbopack)
✓ npm --prefix dungeoneer-frontend run test:unit  -> 6 passed (6 test files), 9 passed (9 tests)
```

---

## Roteiro de Verificação Manual (End-to-End)

1. Inicie os serviços de backend (`api-gateway` na porta `:9090`, `auth-service` e `player-character-service`) e o frontend (`npm --prefix dungeoneer-frontend run dev`).
2. Acesse `http://localhost:3000/characters` sem estar autenticado e confirme o redirecionamento automático para `/login`.
3. Crie uma conta em `/register` ou autentique-se em `/login`; verifique o redirecionamento para `/characters` e a exibição do seu nickname na `Navbar`.
4. Clique em **"Criar Personagem"** (`/characters/new`), preencha os 6 passos (testando a rolagem única `4d6kh3` e o drag-and-drop nos 6 atributos) e finalize em **"Criar Personagem"**.
5. Na ficha (`/characters/[id]`), navegue pelas 8 abas (`Resumo`, `Atributos`, `Perícias`, `Ações`, `Magias`, `Inventário`, `Talentos`, `Notas`), ajuste os PV na aba `Resumo` e salve uma anotação na aba `Notas`.
6. Clique em **"Editar"** (`/characters/[id]/edit`), altere o nome ou PV, salve e confirme a atualização na ficha; por fim, teste a exclusão pelo modal de confirmação.
