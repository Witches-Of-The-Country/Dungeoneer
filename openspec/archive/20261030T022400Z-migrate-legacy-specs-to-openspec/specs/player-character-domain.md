---
id: "player-character-domain"
title: "Player Character Domain & API — Living Specification"
version: "1.0.0"
status: "active"
owners:
  - "@jaca"
scope:
  paths:
    - "player-character-service/**"
resources:
  - "openspec/specs/player-character-data-model.md"
  - "openspec/specs/system-architecture.md"
---

# Player Character Domain & API — Living Specification

> This is the authoritative source of truth for the `player-character-service` domain model, object-oriented design patterns, use cases, SRD seed data scope, and REST API contracts.
> Update this file only after merging a completed change from `openspec/changes/`.

---

## Context & Objectives

The `player-character-service` encapsulates all D&D 5e (2014) and 2024 Revised rules governing player characters, ability scores, class progressions, spellcasting, feats, inventory, and dice resolution. Its core domain layer is pure Java 21 with zero framework dependencies, leveraging GoF and SOLID design patterns to handle complex RPG mechanics cleanly.

### Core Object-Oriented Design Decisions

1. **Composition over Inheritance (`Feat` Strategy)**: Rather than building a rigid class hierarchy for character capabilities, feats and traits are modeled via the `Feat` Strategy interface (`apply(PlayerCharacter pc)`). This enables dynamic injection of capabilities from lineages, classes, backgrounds, and magic items without subclass explosion.
2. **Builder Pattern for Complex Entities (`SpellBuilder`)**: `Spell` entities have many optional attributes (multiple damage dice, conditions, components). `SpellBuilder` standardizes instantiation without telescoping constructors.
3. **Interface Segregation Principle (`SpellcastingStrategy`)**: Spellcasting mechanics are extracted into `SpellcastingStrategy` with dedicated implementations (`FullCaster`, `HalfCaster`, `ThirdCaster`, `PactMagic`, `NonCaster`), keeping martial classes free of unused spellcasting state.
4. **Dependency Inversion Principle (`Target`)**: Spells act upon the `Target` abstraction (`SingleTarget`, `AreaTarget`) rather than concrete map/grid implementations.
5. **Single Responsibility & Value Objects (`Dice`)**: Physical dice are modeled as an immutable `Dice` Value Object (`roll(int quantity)`, `rollCritical(int quantity)`) isolating critical hit and random roll rules.
6. **DRY via `AbilityScores` Abstraction**: Ability scores (`Strength`, `Dexterity`, `Constitution`, `Intelligence`, `Wisdom`, `Charisma`) and modifier calculations (`floor((score - 10) / 2)`) are centralized in `AbilityScores` and composed across PCs and NPCs.
7. **Open/Closed Principle on `Item` Feat Injection**: Equipment and magic items can grant `List<Feat>` to a `PlayerCharacter` when equipped/attuned, extending character capabilities without modifying `PlayerCharacter`.
8. **Cohesive Roll Hierarchy (`DiceRoll` Strategy)**: Mechanical resolutions implement `DiceRoll` (`resolve(AbilityScores stats)`) through specialized classes: `DamageRoll`, `HealRoll`, and `CheckRoll` (unifying attack rolls, ability checks, and saving throws with advantage/disadvantage and proficiency).

---

## Functional Requirements

### Domain Use Cases

```mermaid
flowchart LR
    Player((Jogador))
    DM((Mestre))

    subgraph Dungeoneer [Casos de Uso - Dungeoneer]
        UC1([UC1: Criar/Editar Ficha de Personagem])
        UC2([UC2: Evoluir Personagem / Multiclasse])
        UC3([UC3: Gerenciar Magias e Spell Slots])
        UC4([UC4: Resolver Ações e Rolagens])
        UC5([UC5: Gerenciar Inventário e Itens])
        UC6([UC6: Criar e Gerenciar Campanhas])
        UC7([UC7: Criar e Gerenciar NPCs])
    end

    Player --> UC1
    Player --> UC2
    Player --> UC3
    Player --> UC4
    Player --> UC5

    DM --> UC4
    DM --> UC5
    DM --> UC6
    DM --> UC7
```

- **[RF-01] Create & Manage Character (`UC1`)**: Define base `AbilityScores` (via `STANDARD_ARRAY`, `POINT_BUY` 27-point budget, or server-validated `ROLLED` 4d6-drop-lowest), select `Lineage` and `Background` (injecting corresponding `Feat`s via `RulesEngine`), and initialize the first `ClassProgression`.
  - *Implementation Status*: `PlayerCharacter`, `AbilityScores`, `Ruleset`, `AbilityScoreMethod`, `CreateCharacterUseCase`, outbound repository ports/JPA adapters, and `GlobalExceptionHandler` are implemented; `PlayerCharacterController` (`POST /api/characters`) has failing MockMvc tests written (TDD RED phase) ready for GREEN phase implementation.
- **[RF-02] Level Up & Multiclass (`UC2`)**: Increment levels on an existing `ClassProgression` or add a new `ClassProgression` (validating multiclass ability score prerequisites), select a `Subclass` at the archetype's subclass threshold level, and recalculate proficiency bonus, HP, and `SpellcastingStrategy` slots.
- **[RF-03] Spell Management & Search (`UC3`)**: Query available spells filtered by class, level, school, and name; learn/prepare spells on a character; and track daily spell slot consumption and recovery.
- **[RF-04] Action & Dice Roll Resolution (`UC4`)**: Instantiate and resolve `CheckRoll` (attacks, skill checks, saving throws with advantage/disadvantage/proficiency), `DamageRoll`, and `HealRoll` based on the actor's `AbilityScores`.
- **[RF-05] Inventory & Item Management (`UC5`)**: Add/remove items, track carrying weight, enforce the 3-slot attunement limit, and dynamically apply/remove item-granted `Feat`s.
- **[RF-06] SRD Reference Data Seeding**: Pre-populate SRD reference data via Flyway migrations tagged with `source = 'SRD'` and `ruleset = 'SRD_2014'` or `'SRD_2024'`:

| Category | Count | Source Scope |
|----------|-------|--------------|
| Classes (`archetypes`) | 13 | All SRD base classes |
| Subclasses (`subclasses`) | 13 (1 per class) | SRD only (e.g., Champion, Life Domain) |
| Lineages (`lineages`) | 9 | Human, Elf, Dwarf, Halfling, Gnome, Half-Elf, Half-Orc, Tiefling, Dragonborn |
| Backgrounds (`backgrounds`) | 6 | Acolyte, Criminal, Folk Hero, Noble, Sage, Soldier |
| Spells (`spells`) | ~300 | SRD 5.1 / 5.2 spell list |
| Feats (`feats`) | ~40 | SRD feats |

- **[RF-07] Homebrew Content Extensions**: Allow DMs/users to create custom classes, spells, and feats tagged with `source = 'HOMEBREW'` and scoped by `owner_id`.

---

## Core Domain Class Diagram

```mermaid
classDiagram
    class PlayerCharacter {
        -String name
        -int level
        -int hitPoints
        -AbilityScores abilityScores
        -Set~ClassProgression~ progressions
        -List~Item~ inventory
        -Background background
        -Lineage lineage
        -List~Condition~ conditions
        -List~Feat~ activeFeats
    }

    class AbilityScores {
        <<interface>>
        +getScore(String ability) int
        +getModifier(String ability) int
    }

    class Dice {
        -int faces
        +roll(int quantity) int
        +rollCritical(int quantity) int
    }

    class DiceRoll {
        <<interface>>
        +resolve(AbilityScores stats) int
    }

    class DamageRoll {
        -DamageType damageType
        -int numberOfDice
        -Dice dice
        -boolean addModifier
        -String keyAttribute
    }

    class HealRoll {
        -HealType type
        -int numberOfDice
        -Dice dice
        -boolean addModifier
        -String keyAttribute
    }

    class CheckRoll {
        -boolean advantage
        -boolean disadvantage
        -boolean addProficiency
        -String keyAttribute
    }

    class DamageType {
        <<enumeration>>
        FIRE
        COLD
        LIGHTNING
        THUNDER
        FORCE
        PSYCHIC
        RADIANT
        NECROTIC
        SLASHING
        BLUDGEONING
        PIERCING
    }

    class HealType {
        <<enumeration>>
        HP
        THP
    }

    DiceRoll <|.. DamageRoll
    DiceRoll <|.. HealRoll
    DiceRoll <|.. CheckRoll
    DamageRoll --> DamageType
    HealRoll --> HealType
    DamageRoll --> Dice
    HealRoll --> Dice

    class ClassProgression {
        -Archetype archetype
        -Subclass subclass
        -int level
    }

    class Archetype {
        -String name
        -HitDiceEnum hitDice
        -List~Feat~ classFeats
        -List~Subclass~ availableSubclasses
        -List~Spell~ availableSpells
        -SpellcastingStrategy castingStrategy
    }

    class Subclass {
        -String name
        -String description
        -List~Feat~ subclassFeats
        -List~Spell~ availableSpells
        -SpellcastingStrategy castingStrategy
    }

    class HitDiceEnum {
        <<enumeration>>
        D6
        D8
        D10
        D12
    }

    class Feat {
        <<interface>>
        +getName() String
        +getDescription() String
        +apply(PlayerCharacter pc) void
    }

    class Item {
        <<interface>>
        +getName() String
        +getWeight() float
        +getGrantedFeats() List~Feat~
        +use(PlayerCharacter pc) void
    }

    class Spell {
        <<interface>>
        +getName() String
        +getLevel() int
        +getComponents() List~String~
        +getEffects() List~DiceRoll~
        +getConditionsCaused() List~Condition~
        +cast(PlayerCharacter caster, Target target) void
    }

    class Target {
        <<interface>>
        +applyEffect(Spell spell) void
    }

    class SingleTarget
    class AreaTarget

    class SpellBuilder {
        -String name
        -int level
        -List~String~ components
        -List~DiceRoll~ effects
        -List~Condition~ conditionsCaused
        +setName(String name) SpellBuilder
        +addEffect(DiceRoll roll) SpellBuilder
        +build() Spell
    }

    class SpellcastingStrategy {
        <<interface>>
        +getSpellSlots(int level) int
        +getSpellsKnown() List~Spell~
        +learnSpell(Spell spell) void
    }

    class FullCaster
    class HalfCaster
    class ThirdCaster
    class PactMagic
    class NonCaster

    class Lineage {
        -String name
        -List~Feat~ grantedFeats
    }

    class Background {
        -String name
        -List~Feat~ backgroundFeats
    }

    class Condition {
        -boolean isActive
    }

    SpellcastingStrategy <|.. FullCaster
    SpellcastingStrategy <|.. HalfCaster
    SpellcastingStrategy <|.. ThirdCaster
    SpellcastingStrategy <|.. PactMagic
    SpellcastingStrategy <|.. NonCaster
    
    Archetype "1" o-- "1" SpellcastingStrategy : uses
    Subclass "1" o-- "0..1" SpellcastingStrategy : uses

    FullCaster "1" o-- "*" Spell : spellsKnown
    HalfCaster "1" o-- "*" Spell : spellsKnown
    ThirdCaster "1" o-- "*" Spell : spellsKnown
    PactMagic "1" o-- "*" Spell : spellsKnown

    Archetype "1" o-- "*" Spell : availableSpells
    Subclass "1" o-- "*" Spell : availableSpells
    
    Target <|.. SingleTarget
    Target <|.. AreaTarget
    SpellBuilder ..> Spell : creates
    Spell ..> Target : affects
    Spell "1" *-- "*" DiceRoll : has

    PlayerCharacter "1" *-- "*" ClassProgression : progressions
    PlayerCharacter "1" o-- "1" AbilityScores : stats
    ClassProgression "1" --> "1" Archetype : archetype
    ClassProgression "1" --> "0..1" Subclass : subclass
    Archetype "1" *-- "*" Subclass : availableSubclasses
    Archetype ..> HitDiceEnum : uses

    PlayerCharacter "1" o-- "*" Item : inventory
    PlayerCharacter "1" o-- "1" Background : background
    PlayerCharacter "1" o-- "1" Lineage : lineage
    PlayerCharacter "1" o-- "*" Condition : conditions
    PlayerCharacter "1" o-- "*" Feat : activeFeats

    Item "1" o-- "*" Feat : grantedFeats
    Lineage "1" o-- "*" Feat : grantedFeats
    Background "1" o-- "*" Feat : backgroundFeats
    Archetype "1" o-- "*" Feat : classFeats
    Subclass "1" o-- "*" Feat : subclassFeats
```

---

## REST API Contracts (`player-character-service`)

### Character Management Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/characters` | Create a new character |
| `GET` | `/api/characters` | List all characters |
| `GET` | `/api/characters/{id}` | Get character by ID |
| `PUT` | `/api/characters/{id}` | Update character |
| `DELETE` | `/api/characters/{id}` | Delete character |
| `POST` | `/api/characters/{id}/level-up` | Level up a character |
| `POST` | `/api/characters/{id}/multiclass` | Add a new class to a character |

### Spell Search Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/spells` | List/search spells (query params: `class`, `level`, `school`, `name`) |
| `GET` | `/api/spells/{id}` | Get spell details |

### Reference Data Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/reference/classes` | List all available classes |
| `GET` | `/api/reference/lineages` | List all available lineages |
| `GET` | `/api/reference/backgrounds` | List all available backgrounds |
| `GET` | `/api/reference/feats` | List all available feats |

### Homebrew Content Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/reference/homebrew/spells` | Create homebrew spell |
| `POST` | `/api/reference/homebrew/feats` | Create homebrew feat |
| `POST` | `/api/reference/homebrew/classes` | Create homebrew class |

---

## Non-Functional Requirements

- **[RNF-01] Inbound DTO Validation**: All incoming request DTOs in `application/dto/` must validate fields using Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Min`, `@Max`, `@Valid`).
- **[RNF-02] RFC 7807 `ProblemDetail` Error Mapping**: All domain exceptions must map to deterministic HTTP status codes via `@ControllerAdvice`:

| Domain Exception | HTTP Status | Example |
|-----------------|-------------|---------|
| `CharacterNotFoundException` | `404 Not Found` | Character ID doesn't exist |
| `InvalidAbilityScoreException` | `422 Unprocessable Entity` | Point Buy exceeds 27 points |
| `MulticlassPrerequisiteException` | `422 Unprocessable Entity` | Doesn't meet ability score minimum |
| `SpellSlotExhaustedException` | `409 Conflict` | No spell slots remaining |
| `RulesetMismatchException` | `409 Conflict` | 2014 character trying to join 2024 campaign |

Example `ProblemDetail` payload:
```json
{
  "type": "https://dungeoneer.app/errors/invalid-ability-score",
  "title": "Invalid Ability Score",
  "status": 422,
  "detail": "Point Buy total exceeds the 27-point budget. Current total: 31.",
  "instance": "/api/characters",
  "timestamp": "2026-08-22T03:00:00Z"
}
```

- **[RNF-03] OpenAPI 3.0 & Client Generation**: Endpoints are documented via `springdoc-openapi-starter-webmvc-ui` (`/swagger-ui.html`), with TypeScript clients generated into `dungeoneer-frontend/lib/api/generated/`.

---

## Out of Scope

- Campaign, NPC, and combat encounter management (governed by `openspec/specs/campaign-domain.md`).
- Database column definitions and Flyway SQL schemas (governed by `openspec/specs/player-character-data-model.md`).

---

## Revision History

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0.0 | 2026-10-29 | @jaca | Migrated from `README.md`, `docs/USE-CASES.md`, `docs/spec/SPEC.md`, `docs/STATE.md`, and `docs/spec/res/api-contracts.md` |
