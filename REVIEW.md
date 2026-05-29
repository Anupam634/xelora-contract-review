# Code Review & Assessment — Multiownable Migration

**Reviewer:** Anupam 
**Date:** 2026-05-29  

---

## 1. Task Summary

The task requires two deliverables:

1. **Migrate all `onlyOwner` functions** to a `Multiownable` access-control pattern.
2. **Update the README** with the rationale and approach for the security enhancement.

---

## 2. Codebase Analysis

### Contracts in Scope

| Contract | Functions Using `onlyOwner` | Purpose |
|----------|----------------------------|---------|
| **Recoverable.sol** | `reclaimEther()` | Recovers ether accidentally sent to the contract |
| **TokenVault.sol** | `setAllocation()`, `lock()`, `unlock()`, `transferFor()` | Token distribution & vesting vault management |
| **UpgradeableToken.sol** | `setUpgradeAgent()` | Sets the upgrade agent for token migration |

**Total: 6 functions across 3 contracts** need to be migrated.

### Contracts NOT in Scope

| Contract | Reason |
|----------|--------|
| **Migrations.sol** | Truffle framework contract — uses its own `restricted` modifier, not `onlyOwner`. No change needed. |
| **XeloraCoin.sol** | No `onlyOwner` functions — only defines token constants and the `canUpgrade()` view function. |
| **UpgradeAgent.sol** | Interface contract with no access control. |

### Current Inheritance Chain

```
OpenZeppelin Ownable
    └── Claimable
    └── CanReclaimToken
        └── Recoverable (onlyOwner × 1)
                ├── TokenVault (onlyOwner × 4)
                └── UpgradeableToken (onlyOwner × 1)
                        └── XeloraCoin (+ PausableToken)
```

---

## 3. Identified Risks in Current Design

| Risk | Severity | Description |
|------|----------|-------------|
| **Single point of failure** | 🔴 HIGH | One compromised private key = full control over vault locking, token allocations, upgrades, and fund recovery |
| **Key loss = permanent lockout** | 🔴 HIGH | If the owner loses their key, the vault can never be unlocked and tokens are permanently trapped |
| **No separation of duties** | 🟡 MEDIUM | One address controls all critical operations with zero checks or balances |
| **No redundancy** | 🟡 MEDIUM | No backup mechanism if the owner is unavailable for time-sensitive operations (e.g., unlock after vesting) |

---

## 4. Proposed Solution: `Multiownable` Pattern

### Approach

Create a new `Multiownable.sol` base contract that:

- Maintains a **registry of multiple owner addresses** (mapping + array for enumeration)
- Provides an `onlyMultiowner` modifier to replace `onlyOwner`
- Supports **dynamic add/remove** of owners by existing owners
- Enforces safety invariants:
  - Cannot remove the last owner (prevents permanent lockout)
  - Cannot self-remove (prevents accidental lock-out)
  - Hard cap of 16 owners (bounds gas costs in array operations)

### Migration Plan

```
Multiownable (NEW)
    └── Recoverable (change onlyOwner → onlyMultiowner)
            ├── TokenVault (change onlyOwner → onlyMultiowner × 4)
            └── UpgradeableToken (change onlyOwner → onlyMultiowner × 1)
                    └── XeloraCoin (no changes needed)
```

### Files to Create

| File | Description |
|------|-------------|
| `Multiownable.sol` | New base contract with multi-owner registry, `onlyMultiowner` modifier, `addOwner()`, `removeOwner()`, `getOwners()`, `ownersCount()` |

### Files to Modify

| File | Changes |
|------|---------|
| `Recoverable.sol` | Add `Multiownable` import & inheritance, replace `onlyOwner` → `onlyMultiowner` |
| `TokenVault.sol` | Replace 4× `onlyOwner` → `onlyMultiowner`, update NatSpec docs |
| `UpgradeableToken.sol` | Replace 1× `onlyOwner` → `onlyMultiowner`, update NatSpec docs |
| `READE.md` | Add security rationale, approach documentation, inheritance diagram |

---

## 5. Estimated Delivery Timeline

| Phase | Estimated Time |
|-------|---------------|
| Contract implementation (`Multiownable.sol` + migrations) | 2– days |
| Unit & integration testing | 3–4 hours |
| README documentation | 30 minutes |

