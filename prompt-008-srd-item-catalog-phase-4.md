Focus: DevOps / Docker Compose / Spring Boot / Fullstack Verification
Task: Execute Phase 4: Container Build & End-to-End Verification

## Execution Context
- Active Plan: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/tasks.md`
- Proposal Reference: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/proposal.md`
- Scope: Execute Phase 4 (Tasks 4.1 to 4.3) and finalize the Review Gate. Rebuild and restart `player-character-service` via Docker Compose, verify database migration `V24` execution, validate live REST item queries for normalized categories (`POTION`, `WONDROUS`, `EQUIPMENT`) and rarities, ensure item details work smoothly, consolidate the final cumulative walkthrough in pt-BR, and provide ready-to-use commit commands.

## Implementation Steps
1. **Rebuild & Restart Service Container:**
   - Execute `docker --context default compose up -d --build player-character-service`
   - Inspect container startup logs (`docker --context default compose logs player-character-service --tail 50`) to confirm successful application boot and that Flyway migration `V24__normalize_srd_item_categories_and_seed_catalog.sql` applied cleanly with zero errors.
2. **Verify Live Reference Data Queries:**
   - Query the reference data items API (via API Gateway `http://localhost:9090/api/v1/reference/items` or directly `http://localhost:8080/api/v1/reference/items`):
     - Test category filter `category=POTION` (verify non-zero results including healing potions).
     - Test category filter `category=WONDROUS` (verify non-zero results including Bag of Holding, Cloak of Protection).
     - Test category filter `category=EQUIPMENT` (verify adventuring gear and toolkits).
     - Test rarity filter `rarity=COMMON` (verify both non-magical items and common items are returned).
3. **Verify Item Details & End-to-End Integrity:**
   - Verify that item details endpoint (or modal payload) returns populated attributes, descriptions, and costs in copper coins for seeded items.
4. **Final Review Gate & Test Verification:**
   - Ensure backend test suite passes: `mvn -f player-character-service/backend/pom.xml test`
   - Ensure frontend unit tests and build pass:
     - `npm --prefix dungeoneer-frontend run test:unit`
     - `npm --prefix dungeoneer-frontend run build`

## Quality Gates & Verification
- Run container check and integration verification:
  ```bash
  docker --context default compose up -d --build player-character-service
  docker --context default compose logs player-character-service --tail 30
  ```
- Commit locally:
  ```bash
  git commit -m "chore(infra): rebuild player-character-service with V24 seed and verify item catalog E2E"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared cumulative walkthrough artifact (`openspec/changes/008-srd-item-catalog-seed-and-category-normalization/walkthrough.md` in pt-BR):
1. Document Phase 4 execution: container rebuild status, Flyway migration log validation, and live query verification outputs.
2. Complete the walkthrough as this is the **final phase**:
   - Provide a full consolidated summary of all changes across Phases 1 to 4 (Database Migration, Backend Repositories/Tests, Frontend UI/Filters, and Container E2E Verification).
   - Document verification results from all layers (Maven, Vitest, Next.js build, Docker Compose).
   - Provide an actionable user review checklist and manual smoke-testing guide for UI verification.
   - Outline the final step: readiness for spec consolidation and archival via `sdd-spec-archive`.
