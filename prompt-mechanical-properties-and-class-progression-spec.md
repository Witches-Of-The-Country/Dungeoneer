Focus: Full-Stack / Spring Boot (Java 21 / PostgreSQL / Clean Architecture) & Next.js 16 (React 19 / TypeScript / TailwindCSS 4 / TanStack Query)
Task: Create SDD Change Proposal for Mechanical Properties & Class Progression

## Objective
Structure a formal change proposal and atomic task breakdown for change `mechanical-properties-and-class-progression` following the project's Spec-Driven Development conventions.

## Instructions
1. Activate skill `sdd-spec-create` under `openspec/changes/`.
2. Target change name: `mechanical-properties-and-class-progression`
3. Required proposal elements:
   - Problem & Scope:
     - Items with real mechanical attributes:
       * Weapons: configurable damage dice (e.g., 1d8, 2d6), damage type (slashing, piercing, bludgeoning, fire, etc.), weapon properties (finesse, versatile, two-handed, light, heavy), and attack/damage modifiers.
       * Armor: base Armor Class (AC), AC bonus, armor category/classification (Light, Medium, Heavy, Robe/Clothing), stealth disadvantage flag, and strength requirements.
       * Consumables: mechanical effect descriptions and dice formulas/modifiers (e.g., healing 2d4+2, temporary buff duration/effects).
     - Feats with real mechanical attributes:
       * Ability score increases/modifiers, passive bonuses, granted actions/reactions, advantages, and skill/armor/weapon proficiencies.
     - Class progression & Subclasses:
       * Archetype/Class features mapped by character level (levels 1 to 20).
       * Subclass support with archetype relationship, selection level (e.g., level 3 for standard classes), and subclass-specific features unlocked per level.
     - Character Sheet calculation integration:
       * Equip/unequip mechanics recalculating AC, attack rolls, and damage.
       * Active feat and class feature effects reflected in character attributes and actions.
   - Impacted Areas:
     - Identify impacted database tables, domain models, ports, services, REST controllers, DTOs, and frontend components across `player-character-service` and `dungeoneer-frontend` (invoke `sdd-code-explorer` for codebase mapping).
   - Delta Specs under `openspec/changes/mechanical-properties-and-class-progression/specs/` targeting canonical specs:
     - `player-character-data-model-delta.md`: Flyway schema migrations adding mechanical columns/tables for weapon damage, armor AC/category, consumable effects, feat modifiers, class progression features (1-20), and subclass progression.
     - `player-character-domain-delta.md`: Clean Architecture domain models, value objects, use case ports, DTO contracts, and rules engine strategies for mechanical calculations (AC, damage formulas, feature resolution).
     - `frontend-web-app-delta.md`: UI updates for item cards, character sheet equipment slots, class feature timeline/progression view, subclass picker, and feat selection.
   - Acceptance Criteria & Defensive Security:
     - Formulate testable Given-When-Then criteria covering mechanical item equip/unequip, AC and damage computation, feat modifier application, and level 1-20 class/subclass progression unlock.
     - Input validation via Jakarta Validation on all new DTOs (valid dice notation, positive AC, valid damage types).
     - Shift-left testing strategy detailing unit tests (JUnit 5 / Mockito), integration tests (Testcontainers PostgreSQL), and frontend component/form tests.
   - Atomic Task Breakdown (`tasks.md`):
     - Break down tasks into incremental phases: schema migrations, domain entities/value objects, application use cases & DTOs, REST controllers/mappers, frontend UI integration, and comprehensive test coverage.
4. Once completed, invoke skill `sdd-spec-validate` to audit the proposal before starting any code execution.

Do NOT modify production source code. Present proposal artifact for user review.
