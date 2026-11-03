Focus: Backend / Spring Boot 4 (Java 21) / JPA Persistence & Domain Model
Task: Execute Phase 2: Backend Persistence & Domain Support

## Execution Context
- Active Plan: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/tasks.md`
- Linked Proposal: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/proposal.md`
- Domain Delta: `openspec/changes/009-seed-complete-srd-general-and-origin-feats/specs/player-character-domain-delta.md`
- Scope: Execute Phase 2 only (Tasks 2.1 to 2.3). Inspect and enhance domain models for `MechanicalEffects` to support `SpellChoice`, update `FeatJpaEntity.deserializeEffects` to deserialize full `MechanicalEffects` via Jackson `ObjectMapper` (resolving dropped prerequisites and ability bonus choices), and ensure unit tests in `MechanicalEffectsTest` verify full round-trip JSON serialization and deserialization. Do not proceed to subsequent phases.

## Implementation Steps
1. **Extend Domain Model with `SpellChoice` Record:**
   - Enhance `MechanicalEffects` record to include `SpellChoice` metadata (`cantripsCount`, `spellsCount`, `spellLevel`, `allowedClasses`).
   - Preserve clean architecture invariants (zero framework imports in domain models).
2. **Update JPA Persistence Deserialization:**
   - Update `FeatJpaEntity.deserializeEffects` to parse `MechanicalEffects` fully using Jackson `ObjectMapper`.
   - Ensure all properties (`abilityBonusChoice`, `conditional`, `conditionDescription`, `prerequisiteRules`, `spellChoice`) are deserialized without data loss or fallback truncation.
3. **Domain & Serialization Unit Testing:**
   - Add unit tests in `MechanicalEffectsTest` verifying round-trip JSON serialization and deserialization for feats with `spellChoice`, `prerequisiteRules`, and `abilityBonusChoice`.
   - Verify compatibility with existing seed payloads in `V26__seed_complete_srd_general_and_origin_feats.sql`.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(feats): support spell choice and full mechanical effects deserialization"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/009-seed-complete-srd-general-and-origin-feats/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 3 (Character Creation & Level-Up Validation).
Pause and prompt the user to commit progress before proceeding to Phase 3.
