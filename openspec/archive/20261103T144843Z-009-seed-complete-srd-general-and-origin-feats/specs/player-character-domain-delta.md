# Delta Spec: Player Character Domain — Feat Mechanics & Deserialization

> **Change ID:** `009-seed-complete-srd-general-and-origin-feats`
> **Target Document:** `openspec/specs/player-character-domain.md`

---

## 1. Domain Entities & Value Objects

### `MechanicalEffects` Record Extension
Add support for `SpellChoice` record:
```java
public record SpellChoice(
    int cantripsCount,
    int spellsCount,
    int spellLevel,
    List<String> allowedClasses
) {}
```
Integrated into `MechanicalEffects`:
```java
public record MechanicalEffects(
    Map<String, Integer> abilityBonuses,
    int acBonus,
    int initiativeBonus,
    int speedBonus,
    int hpPerLevelBonus,
    List<String> grantedProficiencies,
    String actionName,
    String actionType,
    String diceFormula,
    Integer usesPerRest,
    String restType,
    AbilityBonusChoice abilityBonusChoice,
    boolean conditional,
    String conditionDescription,
    List<FeatPrerequisite> prerequisiteRules,
    SpellChoice spellChoice
)
```

## 2. Persistence Layer Deserialization

Update `FeatJpaEntity.deserializeEffects` to use Jackson `ObjectMapper` so that:
- `abilityBonusChoice` is restored.
- `prerequisiteRules` are restored and validated during character creation and level-up.
- `spellChoice` is available for future inline accordion components in the character sheet / spellbook.
