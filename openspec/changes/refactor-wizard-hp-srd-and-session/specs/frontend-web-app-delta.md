# Delta Spec: `frontend-web-app`

> **Change ID:** `refactor-wizard-hp-srd-and-session`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Wizard State Machine: Nível Inicial, Rolagem de HP e Multiclasse no Passo 3 (Classe)
- O Passo 1 ("Nome & Regras") passa a conter exclusivamente o Nome do Personagem e a Seleção do Conjunto de Regras (`SRD_2014` vs `SRD_2024`).
- O Passo 3 ("Classe") passa a conter:
  - Seleção da **Classe Principal (Nível 1)**.
  - Definição do Nível Inicial Total (1 a 20).
  - Opções de cálculo de HP (`MAX`, `AVERAGE`, `ROLLED`) e botão de rolagem de dados de vida.
  - **Distribuição Multiclasse (quando Nível Total > 1):** Nível 1 atribuído à Classe Principal; botão para adicionar classes secundárias distribuindo os níveis restantes até a soma fechar no Nível Total.
- **Cache de Rolagem por Classe (`hpRollsByClass`):** O estado do wizard gerencia um mapa `Record<string, HpRollResponse>` indexado por `classId`.
  - Ao selecionar uma classe `A`, exibe-se a rolagem já realizada para `A` ou o botão para rolar dados.
  - Ao alternar para a classe `B`, exige-se nova rolagem para `B` (se ainda não tiver sido feita).
  - Ao retornar para a classe `A`, a rolagem anterior de `A` é recuperada e preservada.
- **Validação de Transição:** O avanço do Passo 3 para o Passo 4 é bloqueado se `level > 1` e método for `ROLLED` e não houver rolagem de dados para a classe atualmente selecionada, ou se a soma dos níveis multiclasse não for igual ao nível total.

### 2. Seleção de Perícias da Classe no Passo 3
- Ao selecionar a Classe Principal no Passo 3, o sistema carrega o catálogo de perícias permitidas pela classe e o número de escolhas (ex: Guerreiro escolhe 2, Ladino escolhe 4, Bardo escolhe 3 quaisquer).
- O jogador seleciona as perícias desejadas.
- No Passo 4 (Antecedente), se o antecedente selecionado conceder uma perícia já escolhida na classe, o sistema sinaliza com um badge e permite trocar a escolha para evitar sobreposição desnecessária.

### 3. Edição Inline de Notas na Ficha
- O componente `NoteCard` ganha botão de ação de edição (ícone de lápis).
- Ao clicar em Editar, o card alterna para modo de formulário inline com campos de título e corpo e botões "Salvar" e "Cancelar".
- O salvamento dispara `useUpdateCharacterNote`, chamando `PUT /api/characters/{id}/notes/{noteId}` e atualizando o cache do React Query sem recarregar a página.

### 4. Controles Numéricos de Ajuste de PV na Ficha
- No Card de Pontos de Vida:
  - Linha 1: Botões `-1 PV` e `+1 PV`.
  - Linha 2 (Ajustes Numéricos por Montante):
    - Bloco da esquerda (Dano): Botão `-` seguido de input numérico (`placeholder="Dano"`). Clicar no `-` subtrai o valor dos PV atuais (mínimo 0) e limpa o campo.
    - Bloco da direita (Cura): Input numérico (`placeholder="Cura"`) seguido de botão `+`. Clicar no `+` adiciona o valor aos PV atuais (limitado ao PV máximo) e limpa o campo.

### 5. Dicionário de Tradução SRD Expandido
- O módulo `lib/srd-translations.ts` é estendido com dicionários para:
  - Traços raciais (`traits_summary`): *Darkvision* -> *Visão no Escuro*, *Fey Ancestry* -> *Ancestralidade Feérica*, *Keen Senses* -> *Sentidos Aguçados*, *Stonecunning* -> *Conhecimento de Pedra*, *Trance* -> *Transe*, etc.
  - Proficiências de equipamentos: armaduras (`LIGHT`, `MEDIUM`, `HEAVY`, `SHIELDS`), armas (`SIMPLE`, `MARTIAL`) e ferramentas.
  - Descrições completas de linhagens e antecedentes.
  - Descrições dos talentos de origem (`originFeatDescription`).
- Todas as queries TanStack em `lib/api/characters.ts` (`getReferenceClasses`, `getReferenceLineages`, `getReferenceBackgrounds`, `getCharacter`) aplicam mapeadores de tradução para todos os campos textuais antes de entregar os dados aos componentes React.

### 6. Guard de Autenticidade e Integridade de Sessão
- O hook `useAuth` passa a executar uma validação de integridade assíncrona no mount (`useEffect`).
- Se houver token e usuário em `localStorage` ou cookies, o cliente verifica se o usuário ainda existe no backend.
- Se a resposta for HTTP 401 Unauthorized ou HTTP 404 Not Found (por exemplo, após limpeza de volumes do banco de dados), a rotina dispara imediatamente `clearAuth()`, expurgando `localStorage`, cookies e emitindo evento `auth-state-change` para sincronizar a navbar com o estado não-autenticado.
- O cliente HTTP `lib/api/client.ts` passa a chamar `clearAuth()` em qualquer interceptação de erro 401.
