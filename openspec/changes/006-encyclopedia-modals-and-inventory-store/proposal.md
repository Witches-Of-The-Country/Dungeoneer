# Proposal: 006-encyclopedia-modals-and-inventory-store

## Summary
Introduce comprehensive detail inspection modals across all Encyclopedia categories (Spells, Feats, Items, Lineages) matching the existing Class modal experience, enable updating of Homebrew creations with backend persistence, resolve input sizing issues in the Homebrew item creator, eliminate character wallet resets and fix generic "Talento" placeholders in character sheets, and provide a full-featured Item Catalog with a Shopping Cart modal supporting automatic coin deduction and conversion.

## Motivation
Currently, players and DMs navigating the Encyclopedia cannot view full descriptions or mechanics for spells, feats, items, or lineages without leaving the page, and cannot edit custom homebrew content once created. Furthermore, creating items with coin costs truncates numbers (e.g. typing "50" shows "0"), character coin balances reset to 0 upon sheet load due to domain immutability defects in `PlayerCharacter.java`, character feats display as generic "Talento" cards due to unhydrated queries, and adding items to the inventory is restricted to an unintuitive dropdown. Resolving these issues provides a seamless RPG management workflow and robust data integrity.

## Scope

### In Scope
- Detail modals for Encyclopedia Spells, Feats, Items, and Lineages with mechanics, lore, and source badges.
- Edit flow and backend `PUT /api/reference/{category}/{id}` endpoints for custom Homebrew entities (`ruleset == HOMEBREW`).
- Layout and styling adjustments in `create-homebrew-dialog.tsx` ensuring multi-digit currency numbers are fully visible.
- Domain fix in `PlayerCharacter.java` (`withItems`, `withFeats`, `withLevelUp`) preserving `currencyPouch` and `avatarUrl`.
- Full character sheet hydration in `CharacterQueryService` and `UpdateCharacterService` via `CharacterSheetHydrator.hydrate(...)`.
- Addition of `source` field to `CharacterFeatDto` and click-to-open detail modal on character sheet Feats cards.
- Rich `ItemCatalogModal` in character sheet Inventory supporting both "Adicionar Direto" (free) and "Loja / Carrinho" (automatic coin deduction and conversion).
- Character summary enhancement: show primary class, primary subclass, and secondary multiclass progressions on character cards in `/characters`.

### Out of Scope
- Modifying official SRD reference data (SRD entities remain immutable).
- Real-time multiplayer synchronization of inventory transactions (sessions use standard REST).
- Offline caching of reference data outside standard TanStack Query caching.

## Proposed Solution
1. **Frontend Encyclopedia Modals**: Create `SpellDetailModal`, `FeatDetailModal`, `ItemDetailModal`, and `LineageDetailModal` using Dark Fantasy RPG UI tokens. Wire `onClick` on `ReferenceCard` across all encyclopedia views.
2. **Homebrew Modification**: Add `PUT /api/reference/{category}/{id}` endpoints in `ReferenceDataController`, handled by `HomebrewService`, restricted to `ruleset == HOMEBREW`. Add an "Editar" button on detail modals for homebrew entities opening pre-filled edit forms.
3. **Currency Input Fix**: Restructure the currency inputs in `create-homebrew-dialog.tsx` into a dedicated responsive grid row with `[appearance:textfield]` to hide browser spin controls and guarantee visible digit width.
4. **Wallet & Feat Hydration Fixes**:
   - Update `PlayerCharacter.java` methods `withItems`, `withFeats`, and `withLevelUp` to carry forward `this.currencyPouch` and `this.avatarUrl`.
   - Inject `CharacterSheetHydrator` into `CharacterQueryService` and `UpdateCharacterService`, executing `hydrate(character)` so feats and items are loaded with full metadata.
5. **Item Catalog & Shopping Cart**: Build `ItemCatalogModal` in `components/character-sheet/` supporting search, category tabs, and dual checkout modes (free loot addition vs. store purchase with gold/copper deduction via `AddCharacterItemCommand.purchaseWithCurrency = true`).

### Impacted Files

| File | Status | Layer | Description |
|------|--------|-------|-------------|
| `PlayerCharacter.java` | Modified | Domain | Preserve `currencyPouch` and `avatarUrl` in immutability methods |
| `CharacterFeatDto.java` | Modified | Application | Add `source` field for origin badge display |
| `CharacterQueryService.java` | Modified | Application | Inject and call `CharacterSheetHydrator.hydrate` |
| `UpdateCharacterService.java` | Modified | Application | Inject and call `CharacterSheetHydrator.hydrate` |
| `ReferenceDataController.java` | Modified | Adapter In | Add `PUT /api/reference/{category}/{id}` endpoints |
| `CreateHomebrewUseCase.java` | Modified | Port In | Declare update methods for homebrew entities |
| `HomebrewService.java` | Modified | Application | Implement update methods enforcing `ruleset == HOMEBREW` |
| `create-homebrew-dialog.tsx` | Modified | Frontend | Widen currency inputs, hide spin buttons, support edit pre-fill |
| `reference-card.tsx` | Modified | Frontend | Support card click triggers across all encyclopedia tabs |
| `spells/page.tsx`, `feats/page.tsx`, `items/page.tsx`, `lineages/page.tsx` | Modified | Frontend | Add modal state and click handlers |
| `SpellDetailModal.tsx`, `FeatDetailModal.tsx`, `ItemDetailModal.tsx`, `LineageDetailModal.tsx` | Created | Frontend | Detailed inspection dialogs with Homebrew edit button |
| `feats-tab.tsx` | Modified | Frontend | Render hydrated feat name and origin badge; open detail modal on click |
| `inventory-tab.tsx` | Modified | Frontend | Replace select dropdown with `ItemCatalogModal` trigger |
| `item-catalog-modal.tsx` | Created | Frontend | Catalog search, category filter, free add, and cart purchase modes |
| `CharacterSummaryResult.java` | Modified | Application | Add `subclassName` and `classes` progression list |
| `CharacterSummaryResponse.java` | Modified | Adapter In | Add `subclassName` and `classes` progression list |
| `app/characters/page.tsx` | Modified | Frontend | Display primary class, subclass, and multiclasses on character card |

## Defensive Security & Validation Considerations
- **Immutability of Official Rules**: All `PUT /api/reference/{category}/{id}` endpoints verify that the target entity has `ruleset == Ruleset.HOMEBREW`. Any attempt to modify SRD entities (`SRD_2014`, `SRD_2024`) will immediately reject with `400 Bad Request` or `403 Forbidden`.
- **Jakarta Validation**: All update request payloads are annotated with `@Valid` and appropriate constraints (`@NotBlank`, `@Min(0)`, `@Size`).
- **Defensive Coin Deduction**: Store checkout via `InventoryService` checks `canAfford(costInCopper)` before debiting. In case of insufficient funds, an RFC 7807 `400 Bad Request` (`CharacterValidationException`) is thrown and no inventory state is mutated.

## Testing Strategy & Shift-Left Plan
- **Domain & Unit Testing (JUnit 5 & Mockito)**:
  - Verify `PlayerCharacter.withItems(...)` and `withFeats(...)` retain existing `currencyPouch` and `avatarUrl`.
  - Verify `HomebrewService.update*` modifies homebrew entities and rejects SRD entities.
  - Verify `CharacterQueryService` returns fully hydrated feat and item names.
- **Web Layer Testing (MockMvc)**:
  - Test `PUT /api/reference/items/{id}` returns `200 OK` on valid homebrew update and `400 Bad Request` on non-homebrew update.
- **Frontend Component Testing (Vitest)**:
  - Verify `ItemCatalogModal` displays items, allows quantity change, and invokes mutation.
  - Verify `ReferenceCard` triggers `onClick` callback.
  - Verify `create-homebrew-dialog.tsx` currency inputs accept and display multi-digit numbers.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Incomplete coin conversion during cart checkout | Medium | Leverage existing, tested `CurrencyPouch.deductWithConversion` domain logic. |
| Overwriting SRD entries during homebrew updates | High | Strict check in `HomebrewService`: `if (entity.getRuleset() != Ruleset.HOMEBREW) throw new IllegalStateException(...)`. |
| Hydration performance overhead on character query | Low | Hydration queries batch load reference items and feats by UUID in single queries. |

## Acceptance Criteria (Given-When-Then)

### AC 1: Encyclopedia Detail Inspection Modals
- **Given** a user is viewing any Encyclopedia tab (Magias, Talentos, Itens, or Linhagens),
- **When** the user clicks on any reference card,
- **Then** a detail modal opens displaying complete mechanics, lore, parameters, and ruleset badges.

### AC 2: Homebrew Entity Editing
- **Given** a user opens the detail modal of an entity with `ruleset == HOMEBREW`,
- **When** the user clicks the "Editar" button, modifies the properties, and submits the form,
- **Then** the updated properties are persisted via `PUT /api/reference/{category}/{id}` and reflected in the UI.

### AC 3: Homebrew Modification Guard on Official Content
- **Given** a user views a detail modal of an official SRD entity (`SRD_2014` or `SRD_2024`),
- **When** the modal renders,
- **Then** the "Editar" button is not shown, and any direct `PUT` request to that ID returns an RFC 7807 error.

### AC 4: Currency Input Legibility in Item Creator
- **Given** a user opens the Homebrew item creation dialog,
- **When** the user enters a multi-digit number (e.g. 50 PO) in any coin input,
- **Then** all entered digits remain visible without horizontal clipping or spinner obstruction.

### AC 5: Character Sheet Wallet Persistence
- **Given** a character with non-zero coins in their pouch,
- **When** the character sheet is loaded or updated (e.g. items or feats added),
- **Then** the coin pouch quantities are preserved and accurately displayed instead of resetting to 0.

### AC 6: Feat Hydration and Detail Modal on Character Sheet
- **Given** a character that possesses chosen feats,
- **When** the user navigates to the "Talentos" tab on the character sheet,
- **Then** each feat displays its real name and origin badge, and clicking the card opens a detail modal with its complete description.

### AC 7: Inventory Item Catalog with Direct Add and Cart Purchase
- **Given** a user viewing the "Inventário" tab on the character sheet,
- **When** the user opens the Item Catalog,
- **Then** they can either add items for free ("Adicionar Direto") or purchase items ("Comprar com Moedas"), automatically deducting the required amount from their coin pouch with conversion.

### AC 8: Character Card Subclass and Multiclass Breakdown
- **Given** a character with a selected subclass and/or secondary multiclass progressions,
- **When** the user views the character card on the `/characters` page,
- **Then** the card displays the primary class with its subclass (e.g. "Guerreiro (Mestre da Batalha)") along with all secondary multiclasses and their levels (e.g. "/ Mago 2").
