# Bugfix Proposal: Remove Unwanted Currency Auto-Consolidation on Item Purchase

> **Change ID:** `006-fix-shop-currency-auto-conversion`
> **Execution Order:** `006`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** Low (Pode postergar)
> **Status:** draft
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Fixes `CurrencyPouch.deductWithConversion` so buying an item from the catalog only deducts the required coins (and breaks the minimum necessary higher coin if change is needed) instead of converting and consolidating the character's entire coin pouch into Platinum/Gold and destroying all Electrum (`EP`) coins.

## Grouped Bugs & Symptoms

- **Bug 1:** Pós-compra de item, tiraria conversão automática; (pouco importante, pode postergar).

## Root Cause Analysis (Causa Raiz)

In `CurrencyPouch.java` (lines 100–113), `deductWithConversion(long costInCopper)` converts the entire wallet to copper (`toTotalCopper()`), subtracts `costInCopper`, and calls `fromCopper(remaining)`.
`fromCopper(remaining)` (lines 35–51) greedily divides the total copper by `1000` (PP), `100` (GP), and `10` (SP), hardcoding `ep = 0`.
As a result:
1. Buying a `1 CP` item when holding `500 CP, 20 SP, 10 EP, 5 GP` involuntarily converts the player's entire wallet into `12 GP, 1 SP, 9 CP`.
2. All Electrum (`EP`) coins are wiped out (`0 EP`) on every purchase.
3. The character sheet already provides an explicit "Consolidar" button (`currency-pouch-card.tsx`) for when the player actually wants to consolidate coins.

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

- Affects `InventoryService.addItem` whenever `purchaseWithCurrency == true`.
- Explicit consolidation (`CurrencyPouch.consolidate()`) should remain available for the dedicated "Consolidar" button on the sheet, while item purchase uses non-destructive deduction.

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Table | Discrepancy Identified | Adaptation / Migration Action |
|-------------------------|------------------------|-------------------------------|
| `player_character.characters` (`currency_cp`, `sp`, `ep`, `gp`, `pp`) | Existing coin balances stored in 5 integer columns | No schema migration needed. Refactoring `CurrencyPouch.deductWithConversion` preserves existing coin balances across all 5 denominations on future purchases. |

## Scope

### In Scope
- Refactor `CurrencyPouch.deductWithConversion(long costInCopper)` to deduct from existing coins without consolidating untouched denominations:
  1. Spend exact matching denominations first (or lower denominations up to the cost).
  2. If a higher-denomination coin must be broken to make change, break only the single smallest sufficient higher coin and return change in standard denominations (`GP`/`SP`/`CP`), leaving all remaining `PP`, `GP`, `EP`, `SP`, and `CP` untouched.
- Update `CurrencyPouchTest.java` with comprehensive multi-denomination purchase tests.

### Out of Scope
- Changing D&D 5e exchange rates (`1 PP = 10 GP = 20 EP = 100 SP = 1000 CP`).

## Proposed Solution

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/.../domain/model/CurrencyPouch.java` | Modify | Replace greedy `fromCopper(remaining)` in `deductWithConversion` with non-consolidating coin deduction |
| `player-character-service/.../domain/model/CurrencyPouchTest.java` | Modify | Add unit tests verifying untouched denominations (including `EP`, `CP`, `SP`, `GP`, `PP`) are preserved after item purchase |

## Acceptance Criteria

- [ ] **Given** a character with `100 CP, 20 SP, 10 EP, 10 GP, 2 PP` **When** they buy an item costing `5 GP` (`500 CP`) **Then** their resulting balance is `100 CP, 20 SP, 10 EP, 5 GP, 2 PP` (no auto-consolidation of CP/SP/EP into GP/PP).
- [ ] **Given** a character with `0 SP` and `5 GP` **When** they buy an item costing `5 SP` (`50 CP`) **Then** only `1 GP` is broken into `10 SP` to pay the `5 SP`, resulting in `5 SP` and `4 GP`.
- [ ] **Given** a character with `1 GP` **When** they try to buy an item costing `2 GP` **Then** `CharacterValidationException` ("Insufficient funds") is thrown.

## Defensive Security & Validation Considerations

### Input Validation
- Validates `costInCopper >= 0` and `toTotalCopper() >= costInCopper` before modifying balances.

### Secret Isolation
- N/A.

### Authorization & Authentication Impact
- N/A.

### Other Security Risks
- Prevents integer overflow/underflow by using `long` arithmetic and validating non-negative coin counts in `CurrencyPouch` constructor.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Deduct 5 GP from mixed pouch (`100 CP, 20 SP, 10 EP, 10 GP, 2 PP`) | Unit (`CurrencyPouchTest`) | Only GP decreases by 5; `CP`, `SP`, `EP`, and `PP` remain identical |
| TR-02 | Deduct 5 SP from `1 GP` (`0 SP`) | Unit (`CurrencyPouchTest`) | Breaks `1 GP` → leaves `5 SP, 0 GP` |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Purchase cost exceeds total pouch value in copper | Unit (`CurrencyPouchTest`) | Throws `CharacterValidationException` |

### Test Commands
- **Backend tests:** `mvn clean test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Complex multi-coin payment when player has no single coin large enough but enough combined small coins | Low | Low | Deduct greedily from required denomination, then smaller coins, then break minimal higher coin if needed |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document non-consolidating currency deduction on item purchase in `CurrencyPouch`.
