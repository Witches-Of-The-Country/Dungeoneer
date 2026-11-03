Focus: Backend / Spring Boot 4 (Java 21) / Application Services, DTOs & Persistence
Task: Execute Phase 2: Application Services, Hydration, DTO Validation & Integration Tests

## Execution Context
- Active Plan: `openspec/changes/add-homebrew-feat-charges/tasks.md`
- Linked Proposal: `openspec/changes/add-homebrew-feat-charges/proposal.md`
- Cumulative Walkthrough: `openspec/changes/add-homebrew-feat-charges/walkthrough.md`
- Scope: Execute Phase 2 only. Implement dynamic max charges hydration in `CharacterSheetHydrator`, initialize charges in `CreateCharacterService` and `LevelUpService`, update DTOs, cross-field validation in `HomebrewService`, JSONB persistence compatibility, and MockMvc/Integration tests (TP-03, TE-03, TE-04). Do not proceed to frontend UI.

## Implementation Steps
1. **Character Sheet Hydration & Service Initialization:**
   - Update `CharacterSheetHydrator` to dynamically recalculate feat `maxCharges` based on character level and ability scores on each hydration, clamping `currentCharges` to the computed maximum.
   - Update `CreateCharacterService` to initialize charges when feats are granted during character creation.
   - Update `LevelUpService` to initialize charges for newly granted feats during level progression.
2. **DTO & API Contract Alignment:**
   - Update `CreateHomebrewFeatRequest` with Jakarta Validation annotations (`@NotNull`, `@Min(1)`, `@Max(99)`) for charge configuration fields.
   - Update `FeatDto` and `CharacterFeatDto` to expose dynamic max charges, current charges, and rest recovery metadata.
3. **Homebrew Validation & Persistence Compatibility:**
   - Implement cross-field validation in `HomebrewService` (e.g., verifying `chargeAbility` is provided when formula is `ABILITY_MODIFIER`, and positive fixed numbers when type is `FIXED`), throwing RFC 7807 400 ProblemDetail on invalid input.
   - Verify `FeatJpaEntity` JSONB mapping correctly persists and deserializes charge configuration fields without requiring database migrations, ensuring backward compatibility with legacy feat entries.
4. **Integration & API Tests:**
   - Add MockMvc and integration tests covering:
     - TP-03: Create homebrew feat with custom charge configuration returns 201 Created and persists valid JSONB.
     - TE-03: Invalid configurations (missing ability for `ABILITY_MODIFIER`, invalid fixed numbers) return 400 ProblemDetail.
     - TE-04: Existing legacy feats without new JSONB fields hydrate safely with default values.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(feats): integrate charge calculation into hydration, homebrew api, and persistence"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/add-homebrew-feat-charges/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 3 (Frontend UI, Dialog & Vitest).
Pause and prompt the user to commit progress before proceeding to Phase 3.
