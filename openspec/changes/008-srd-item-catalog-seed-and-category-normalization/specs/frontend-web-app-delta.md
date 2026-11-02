# Frontend Web App Delta: 008-srd-item-catalog-seed-and-category-normalization

## 1. Item Encyclopedia (`app/encyclopedia/items/page.tsx`)
- Rarity Filter Options:
  - `ALL` -> Todos
  - `COMMON` -> Comum
  - `UNCOMMON` -> Incomum
  - `RARE` -> Raro
  - `VERY_RARE` -> Muito Raro
  - `LEGENDARY` -> Lendário
  - `ARTIFACT` -> Artefato
- Category Filter Tabs:
  - `ALL` -> Todos
  - `WEAPON` -> Armas
  - `ARMOR` -> Armaduras
  - `POTION` -> Poções
  - `WONDROUS` -> Itens Maravilhosos
  - `RING` -> Anéis
  - `SCROLL` -> Pergaminhos
  - `WAND` -> Varinhas
  - `ROD` -> Bastões
  - `STAFF` -> Cajados
  - `EQUIPMENT` -> Equipamento / Ferramentas

## 2. Character Sheet Item Catalog & Store (`components/character-sheet/item-catalog-modal.tsx`)
- Aligns category dropdown and rarity dropdown with the same canonical enums (`WEAPON`, `ARMOR`, `POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`).
- Displays loaded items in their correct tabs.
