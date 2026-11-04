Focus: Full-Stack / Spring Boot (Java 21 / PostgreSQL) & Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Create SDD Change Proposal for Add Homebrew Creation

## Objective
Structure a formal change proposal and atomic task breakdown for change `add-homebrew-creation` following the project's Spec-Driven Development conventions.

## Instructions
1. Activate skill `sdd-spec-create` under `openspec/changes/` to complete the active change `add-homebrew-creation`.
2. Target change name: `add-homebrew-creation`
3. Required proposal elements:
   - Problem & Scope:
     - Document DM (Dungeon Master) capabilities to create custom reference data (Archetypes, Spells, Feats, Items, Lineages) tagged with ruleset `HOMEBREW`.
     - Owner isolation: homebrew content is strictly visible to its creator (`user_id`). Future campaign sharing remains out of scope.
   - Impacted Areas:
     - Verify and refine the impacted files mapping across `api-gateway`, `player-character-service`, and `dungeoneer-frontend` (invoke `sdd-code-explorer` if further mapping is needed).
   - Delta Specs under `openspec/changes/add-homebrew-creation/specs/` targeting canonical specs:
     - `player-character-data-model-delta.md`: Schema updates (Flyway migration extending `ruleset` check constraint to include `HOMEBREW`, adding nullable `owner_id` / `user_id` column, and composite indexes for visibility queries).
     - `player-character-domain-delta.md`: Domain ruleset enum expansion, homebrew creation use case ports, DTO contracts, and repository query filters (`owner_id IS NULL OR owner_id = :userId`).
     - `system-architecture-delta.md`: API Gateway route protection for `POST /api/reference/**`, DM role requirement, and `X-User-Id` / `X-User-Roles` header propagation.
     - `frontend-web-app-delta.md`: UI components for DM-only "Criar novo" dialogs, form validation, and TanStack Query mutations.
   - Acceptance Criteria & Defensive Security:
     - Formulate testable Given-When-Then criteria covering creation by DM, role-based rejection for non-DM users, visibility isolation per creator, and input validation.
     - Input validation via Jakarta Validation on all DTOs and sanitization against XSS in description fields.
     - Shift-left testing strategy detailing unit, integration (Testcontainers), and frontend tests.
   - Atomic Task Breakdown (`tasks.md`):
     - Replace placeholders with atomic, checkable tasks for migrations, domain, application ports/services, REST controllers, gateway filtering, frontend dialogs/mutations, and automated test coverage.
4. Once completed, invoke skill `sdd-spec-validate` to audit the proposal before starting any code execution.

Do NOT modify production source code. Present proposal artifact for user review.
