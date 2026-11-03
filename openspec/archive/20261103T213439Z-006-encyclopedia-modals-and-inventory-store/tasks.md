# Tasks: 006-encyclopedia-modals-and-inventory-store

## Phase 1: Backend Domain & Hydration Bug Fixes (Critical Fixes)
- [x] 1.1 Fix `PlayerCharacter.java` methods `withItems`, `withFeats`, and `withLevelUp` to preserve `this.currencyPouch` and `this.avatarUrl`.
- [x] 1.2 Inject `CharacterSheetHydrator` into `CharacterQueryService` and `UpdateCharacterService`, invoking `sheetHydrator.hydrate(...)` so feats and items are fully populated.
- [x] 1.3 Add `source` field to `CharacterFeatDto` and map it from `choicesJson` or Feat definition in `CharacterSheetHydrator`.
- [x] 1.4 Update `CharacterSummaryResult`, `CharacterSummaryResponse`, `PlayerCharacterWebMapper`, and `CharacterQueryService.listCharacters` to include `subclassName` and class progressions list (for multiclass and subclass card display).
- [x] 1.5 Add unit tests in `PlayerCharacterTest` verifying `currencyPouch` and `avatarUrl` are retained across `withItems` / `withFeats`.
- [x] 1.6 Add unit tests in `CharacterQueryServiceTest` verifying hydrated feats, items, and multiclass summary.

## Phase 2: Backend Homebrew Update Endpoints & Security Checks
- [x] 2.1 Add `updateSpell`, `updateFeat`, `updateItem`, `updateArchetype`, `updateLineage`, and `updateSubclass` to `CreateHomebrewUseCase` and `HomebrewService`.
- [x] 2.2 Enforce security rule: verify `entity.getRuleset() == Ruleset.HOMEBREW` and throw `CharacterValidationException` if an attempt is made to mutate non-homebrew rulesets.
- [x] 2.3 Add `PUT /api/reference/{category}/{id}` endpoints to `ReferenceDataController` with `@Valid` DTO parameters.
- [x] 2.4 Add unit and web tests in `ReferenceDataControllerTest` and `HomebrewServiceTest` testing both successful homebrew updates and rejected SRD modification attempts.
- [x] 2.5 Run backend unit tests (`mvn test`) and restart backend container.

## Phase 3: Frontend Homebrew Currency Input Sizing Fix
- [x] 3.1 Refactor currency cost inputs in `create-homebrew-dialog.tsx` into a dedicated responsive row with hidden webkit spinners (`[appearance:textfield]`) and ample column width.
- [x] 3.2 Verify multi-digit values (e.g. 50 PO) render without truncation or clipping.

## Phase 4: Frontend Character Sheet Feats Tab & Modals
- [x] 4.1 Update `feats-tab.tsx` to display hydrated feat name, origin badge (Lineage, Archetype/Class, Background, Custom), and full mechanics summary.
- [x] 4.2 Add click interaction on feat cards opening a `FeatDetailModal`.
- [x] 4.3 Verify currency update in `currency-pouch-card.tsx` stays saved after update without resetting to 0.
- [x] 4.4 Update character cards in `app/characters/page.tsx` to display subclass and multiclass progression breakdown.

## Phase 5: Frontend Encyclopedia Detail Modals & Homebrew Edit
- [x] 5.1 Implement `SpellDetailModal.tsx` for spell school, casting time, range, components, duration, damage rolls, and upcast descriptions.
- [x] 5.2 Implement `FeatDetailModal.tsx` for prerequisites, stat bonuses, and mechanical effects.
- [x] 5.3 Implement `ItemDetailModal.tsx` for type, rarity, weight, cost, attunement, and item mechanics.
- [x] 5.4 Implement `LineageDetailModal.tsx` for traits, speed, size, and racial abilities.
- [x] 5.5 Connect click handlers on `ReferenceCard` across Spells, Feats, Items, and Lineages pages.
- [x] 5.6 Add "Editar" button in modals when `ruleset === 'HOMEBREW'`, wiring it to edit dialog and `PUT` API.

## Phase 6: Frontend Inventory Item Catalog & Shopping Cart Modal
- [x] 6.1 Create `item-catalog-modal.tsx` with search, category filtering (Armas, Armaduras, Consumíveis, etc.), and rarity filter.
- [x] 6.2 Add "Adicionar Direto" (free add with quantity selector).
- [x] 6.3 Add "Loja / Carrinho" mode displaying unit and total price, comparing against character's current coin pouch, and checking out with `purchaseWithCurrency: true`.
- [x] 6.4 Replace the old dropdown in `inventory-tab.tsx` with the new catalog modal trigger.

## Phase 7: Verification & Build
- [x] 7.1 Run frontend unit tests (`npm run test:unit`).
- [x] 7.2 Run frontend production build (`npm run build`).
- [x] 7.3 Run backend tests (`mvn test`).

## Review Gate
- [x] All 7 acceptance criteria verified via tests and visual checks.
- [x] Backend line coverage >= 80% on modified domain and application services.
- [x] Frontend builds without TypeScript or ESLint errors.
- [x] No regression on existing character creation, level up, or dice roll flows.
- [x] Conventional commit prepared for user approval.
