# Delta Spec: `frontend-web-app`

> **Change ID:** `fix-character-sheet-bugs`
> **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Guard de Rolagem de Vida no Wizard

O passo de rolagem de vida no wizard de criação de personagem deve:
- Verificar se `character.class !== null` antes de habilitar o botão
- Exibir `Tooltip` com mensagem "Selecione uma classe primeiro" enquanto desabilitado
- Passar o `hit_die` da classe selecionada para o hook de rolagem

### 2. Mapa de Traduções SRD PT-BR

Criar `src/lib/srd-translations.ts` com exportações:
```ts
export const CLASS_NAMES_PT: Record<string, string> = { ... }
export const RACE_NAMES_PT: Record<string, string> = { ... }
export const BACKGROUND_NAMES_PT: Record<string, string> = { ... }
export const SPELL_NAMES_PT: Record<string, string> = { ... }
export const SKILL_NAMES_PT: Record<string, string> = { ... }

export function translateSrd(key: string, map: Record<string, string>): string {
  return map[key] ?? key; // fallback para nome original
}
```

Aplicar nos TanStack Query `select` transforms antes de retornar dados para os componentes.

### 3. Novos Componentes: Notas Incrementais

#### `NoteCard`
```tsx
interface NoteCardProps {
  title: string;
  body: string;
  createdAt: string; // ISO 8601
}
```
Exibe título em destaque, corpo do texto e data formatada em PT-BR.

#### `AddNoteForm`
```tsx
interface AddNoteFormProps {
  characterId: string;
  onSuccess: () => void;
}
```
Formulário com campos `title` (obrigatório, max 255) e `body` (obrigatório, max 10.000). Usa `useMutation` do TanStack Query para `POST /characters/{id}/notes`.

#### Integração na ficha
A seção de notas exibe `<AddNoteForm>` no topo seguido de lista de `<NoteCard>` em ordem cronológica reversa.

### 4. Correção de Drag & Drop

O `@dnd-kit/core` já está instalado e configurado no projeto. O atraso (lag) perceptível ocorre devido à classe CSS `transition-all` no componente `DraggableScoreChip` em `dungeoneer-frontend/app/characters/new/page.tsx:127`. Essa classe força o navegador a interpolar via animação CSS o `transform: translate3d(...)` a cada micro-movimento do ponteiro:
- Remover a classe `transition-all` durante o estado ativo de arraste (`isDragging ? "" : "transition-all"`) ou utilizar o componente `<DragOverlay>` fornecido pelo `@dnd-kit/core` para desacoplar a renderização do elemento original e do clone arrastado.
- Garantir resposta imediata (< 16ms) do cursor durante movimentos rápidos do mouse.
