# CERTORA BEST PRACTICES & TECHNIQUES

> **Extracted from official Certora Tutorials**  
> **Integrated with your framework phases**

---

## TABLE OF CONTENTS

1. [Property Discovery Techniques](#1-property-discovery-techniques)
2. [Counter-Example Investigation](#2-counter-example-investigation)
3. [Invariant Design Patterns](#3-invariant-design-patterns)
4. [Harness Best Practices](#4-harness-best-practices)
5. [Loop Handling](#5-loop-handling)
6. [Common Pitfalls](#6-common-pitfalls)

---

# 1. PROPERTY DISCOVERY TECHNIQUES

## 1.1 The Critical Importance of Property Discovery

> **From Certora Tutorial Lesson 06:**
> 
> "Coming up with meaningful properties is the most challenging part of the work. Without the ability to identify and express meaningful properties, all your technical knowledge with the tool is worthless."

### The Four Fatal Mistakes

| Mistake | Consequence | How to Avoid |
|---------|-------------|--------------|
| **Wrong Property** | Will never be proved, unhelpful CEs | Use plain English first, validate logic |
| **Partial Coverage** | Many bugs won't be detected | Use comprehensive categorization |
| **Duplicate Property** | Wasted verification time | Track property IDs, review overlaps |
| **Mimicking Implementation** | Verifies code works wrong | Write from spec, not from code |

### The Mimicking Implementation Anti-Pattern

```
❌ WRONG: Copying requires from implementation
```solidity
// Implementation
function withdraw(uint amount) external {
    require(balance[msg.sender] >= amount);  // ← DON'T copy this
    ...
}
```

```cvl
// BAD spec (mimics implementation)
rule withdraw_check(uint amount) {
    require balance[msg.sender] >= amount;  // ← Copying the bug!
    ...
}
```

```
✅ CORRECT: Define from specification
```cvl
// GOOD spec (from intended behavior)
rule withdraw_correctness(uint amount) {
    uint balBefore = balance[msg.sender];
    withdraw(amount);
    uint balAfter = balance[msg.sender];
    
    // Specify WHAT should happen, not HOW it's implemented
    assert balAfter == balBefore - amount, "Balance should decrease by amount";
    assert amount <= balBefore, "Cannot withdraw more than balance";
}
```

## 1.2 Iterative Property Discovery

> **Key Insight from Certora:**
> 
> "The process of coming up with properties is not done once you start verification. It is an iterative process. While writing rules, you will better understand the system through violations."

### The Discovery Cycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Phase 2: Initial Properties                                           │
│   ├── Study interface & comments                                        │
│   ├── Define in plain English                                           │
│   └── Categorize (Valid State / Transition / System / Threat)           │
│                                                                          │
│                     ▼                                                    │
│                                                                          │
│   Phase 7: Write CVL                                                    │
│   ├── Formalize properties                                              │
│   ├── Run verifications                                                 │
│   └── Get violations or passes                                          │
│                                                                          │
│                     ▼                                                    │
│                                                                          │
│   Iteration: Learn from Results                                         │
│   ├── Understand violations → discover NEW properties                   │
│   ├── See passes → realize MISSING properties                           │
│   └── Refine understanding → UPDATE properties                          │
│                                                                          │
│                     ▼                                                    │
│                                                                          │
│   Back to Phase 2: Refined Properties                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1.3 Property Prioritization Framework

> **From Auction Demonstration:**

After discovering properties, **prioritize them by impact:**

### Priority Matrix

| Priority | Criteria | Example |
|----------|----------|---------|
| **HIGH** | Loss of funds, DoS, auction rigging | "Winner cannot claim prize" |
| **HIGH** | Value creation from nothing | "Mint without backing" |
| **HIGH** | Privilege escalation | "Non-admin can pause" |
| **MEDIUM** | Solvency (when no withdrawal exists) | "Total supply tracking" |
| **LOW** | Single function correctness | "mint() increases balance" |
| **LOW** | Easy to check manually | Simple arithmetic |

### Prioritization Template

```markdown
## Property Prioritization

### HIGH PRIORITY
**[ID]. [Property Name]**
- **Impact:** [Loss of funds / DoS / Manipulation]
- **Reasoning:** [Why this is critical]
- **Dependencies:** [What other properties depend on this]

### MEDIUM PRIORITY
**[ID]. [Property Name]**
- **Impact:** [Accounting issue / Temporary inconsistency]
- **Reasoning:** [Why this is important but not critical]

### LOW PRIORITY
**[ID]. [Property Name]**
- **Impact:** [Local function behavior]
- **Reasoning:** [Can be checked by other means]
```

---

# 2. COUNTER-EXAMPLE INVESTIGATION

## 2.1 The Investigation Process

> **From Lesson 02: Investigate Violations**

### Step-by-Step CE Investigation

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   STEP 1: Run Entire Spec First                                         │
│   ─────────────────────────────                                         │
│   certoraRun config.conf                                                 │
│                                                                          │
│   → See which rules fail                                                │
│   → Get overview of issues                                              │
│                                                                          │
│   STEP 2: Focus on One Rule                                             │
│   ──────────────────────────                                            │
│   certoraRun config.conf --rule specific_rule_name                       │
│                                                                          │
│   → Saves run time                                                      │
│   → Cleaner output                                                      │
│                                                                          │
│   STEP 3: Analyze Call Trace                                            │
│   ───────────────────────────                                           │
│   • Check storage values before/after                                   │
│   • Check arguments passed                                              │
│   • Check return values                                                 │
│   • Follow execution path                                               │
│                                                                          │
│   STEP 4: Identify Deviation                                            │
│   ───────────────────────────                                           │
│   • Where does execution differ from spec?                              │
│   • Is it a spec bug or contract bug?                                   │
│   • Is the property too strong or implementation wrong?                 │
│                                                                          │
│   STEP 5: Fix and Verify                                                │
│   ─────────────────────────                                             │
│   • Fix the issue (spec OR contract)                                    │
│   • Re-run to confirm fix                                               │
│   • Document the fix                                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Call Trace Analysis Tips

### Tip 1: Storage Information

> **From Tutorial:**
> "Call traces contain information regarding the storage, arguments and return value."

**What to check in storage:**
- Initial values before function call
- Changed values after function call
- Values that should have changed but didn't
- Values that changed unexpectedly

### Tip 2: Breaking Down Complex Expressions

> **From Tutorial:**
> "Try breaking complex expressions to achieve code readability and a more simplified call trace."

```cvl
// ❌ Complex (hard to debug)
assert balanceOf(user) + debt[user] <= totalSupply() + totalDebt();

// ✅ Broken down (easier call trace)
uint256 userTotal = balanceOf(user) + debt[user];
uint256 systemTotal = totalSupply() + totalDebt();
assert userTotal <= systemTotal, "User total exceeds system total";
```

## 2.3 Bug Documentation Format

When you find a bug via CE, document it:

```markdown
### Bug Found: [Rule Name] Violation

**File:** `Contract.sol:LineNumber`

**Issue:** 
[One-sentence description]

**Fix:**
```solidity
// OLD (buggy)
require(a > b);

// NEW (fixed)
require(b > a);
```

**Note:** 
The require checked `a > b`, when it should've checked `b > a`.
This allowed [exploit description].
```

---

# 3. INVARIANT DESIGN PATTERNS

## 3.1 Inductive Reasoning Prerequisites

> **From Lesson 07:**
> 
> "Read about preconditions and postconditions to understand how and when to use them."

### Specification Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   PRECONDITION  → OPERATION → POSTCONDITION                            │
│                                                                          │
│   "Requires X    Method M     Ensures Y"                                │
│    to be true"   executes     becomes true"                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Invariant = Always True Postcondition

```cvl
invariant myInvariant()
    condition  // ← This must ALWAYS be true
{
    preserved {
        // Preconditions for operations that preserve the invariant
        requireInvariant otherInvariant();
    }
}
```

## 3.2 Ghost-Based Invariants (Monotonicity Patterns)

> **From Lesson 15: Harness Exercises**

### Pattern 1: Monotonicity of Counters

```cvl
/// @title Points are non-decreasing
/// @notice Total distributed points can only increase or stay same
ghost mathint totalPointsGhost {
    init_state axiom totalPointsGhost == 0;
}

hook Sstore points uint256 newPoints (uint256 oldPoints) {
    totalPointsGhost = totalPointsGhost + (newPoints - oldPoints);
}

invariant pointsMonotonicity()
    totalPointsGhost >= 0
```

### Pattern 2: Per-Entity Monotonicity

```cvl
/// @title Each contender's points are non-decreasing
ghost mapping(address => mathint) contenderPointsGhost {
    init_state axiom forall address c. contenderPointsGhost[c] == 0;
}

hook Sstore contenderPoints[KEY address c] uint256 newPts (uint256 oldPts) {
    contenderPointsGhost[c] = contenderPointsGhost[c] + (newPts - oldPts);
}

invariant contenderMonotonicity(address contender)
    contenderPointsGhost[contender] >= 0
```

### Pattern 3: Relationship Between Aggregates

```cvl
/// @title Single entity < Total
/// @notice Any contender's points < total points
invariant contenderLessThanTotal(address c)
    contenderPointsGhost[c] <= totalPointsGhost
    {
        preserved {
            requireInvariant pointsMonotonicity();
        }
    }
```

### Pattern 4: Conservation Law

```cvl
/// @title Points Conservation
/// @notice Total points = number of voters * points per vote
invariant pointsConservation()
    totalPointsGhost == numVotersGhost * 6
    {
        preserved {
            requireInvariant pointsMonotonicity();
            requireInvariant votersMonotonicity();
        }
    }
```

## 3.3 Preserved Block Best Practices

> **From Lesson 08: Working With Invariants**

### Structure of Preserved Blocks

```cvl
invariant myInvariant()
    condition
{
    preserved with (env e) {
        // Preconditions that must hold for ALL operations
        require e.msg.sender != 0;
        require e.block.timestamp < 2^40;
    }
    
    preserved specificFunction(uint256 x) with (env e) {
        // Preconditions specific to this function
        require x < MAX_VALUE;
        requireInvariant dependency1();
        requireInvariant dependency2();
    }
}
```

### When to Use Preserved Blocks

| Use Case | Syntax |
|----------|--------|
| Global constraints for all functions | `preserved with (env e) { ... }` |
| Function-specific constraints | `preserved funcName(args) with (env e) { ... }` |
| Invariant dependencies | `requireInvariant otherInvariant();` |
| Realistic bounds | `require value < MAX_REALISTIC;` |

---

# 4. HARNESS BEST PRACTICES

## 4.1 When to Use Harnesses

> **From Lesson 15: Harness**

### Harness Use Cases

| Reason | Example |
|--------|---------|
| Expose private state | Getter for internal variable |
| Simplify complex external contracts | DummyERC20 instead of full OpenZeppelin |
| Add helper functions for verification | Sum of mapping values |
| Bypass constructor constraints | Set initial state directly |

## 4.2 Harness Design Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./OriginalContract.sol";

/**
 * @title ContractHarness
 * @notice Verification harness for OriginalContract
 * @dev Exposes internal state and adds helper functions
 */
contract ContractHarness is OriginalContract {
    
    // ═══════════════════════════════════════════════════════════════
    // GETTERS FOR INTERNAL STATE
    // ═══════════════════════════════════════════════════════════════
    
    /// @notice Expose internal variable
    function getInternalVariable() external view returns (uint256) {
        return internalVariable;
    }
    
    // ═══════════════════════════════════════════════════════════════
    // HELPER FUNCTIONS FOR VERIFICATION
    // ═══════════════════════════════════════════════════════════════
    
    /// @notice Sum of all user balances (for solvency checks)
    function sumOfBalances(address[] memory users) external view returns (uint256) {
        uint256 sum = 0;
        for (uint i = 0; i < users.length; i++) {
            sum += balances[users[i]];
        }
        return sum;
    }
    
    // ═══════════════════════════════════════════════════════════════
    // STATE MANIPULATION FOR TESTING
    // ═══════════════════════════════════════════════════════════════
    
    /// @notice Set internal state (for specific test scenarios)
    function setInternalState(uint256 value) external {
        internalVariable = value;
    }
}
```

---

# 5. LOOP HANDLING

## 5.1 Loop Unrolling Strategy

> **From Lesson 11: Loops**

### The Problem

Loops create exponential path explosion for SMT solvers. Certora uses **loop unrolling** as an approximation.

### Configuration Flags

| Flag | Purpose | When to Use |
|------|---------|-------------|
| `--loop_iter N` | Unroll loops N times | Always set (default: 1) |
| `--optimistic_loop` | Assume loop completes correctly | When loop_iter < actual iterations |

### Safe Loop Handling

```conf
{
    "files": ["Contract.sol"],
    "verify": "Contract:spec.spec",
    "loop_iter": "3",           // Unroll 3 times
    "optimistic_loop": true     // Assume correct completion
}
```

### ⚠️ Danger of Wrong Flagging

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   SCENARIO: Loop with bug that manifests on 4th iteration               │
│                                                                          │
│   WITH: --loop_iter 3 --optimistic_loop                                 │
│   RESULT: ✅ PASS (FALSE POSITIVE! Bug not detected)                   │
│                                                                          │
│   WITH: --loop_iter 5                                                   │
│   RESULT: ❌ FAIL (Bug detected correctly)                             │
│                                                                          │
│   LESSON: Set loop_iter high enough to cover realistic scenarios        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Best Practice

```markdown
## Loop Iteration Guidelines

1. **Analyze loop bounds:** How many iterations in realistic scenarios?
2. **Set loop_iter conservatively:** At least realistic max + 1
3. **Document choice:** Explain why N iterations is sufficient
4. **Test sensitivity:** Try higher values to verify no new violations

Example:
```json
{
    "loop_iter": "5",  // Max 4 voters expected, +1 for safety
    "optimistic_loop": true
}
```

Justification: Contract limits to 4 voters, 5 iterations covers all cases.
```

---

# 6. COMMON PITFALLS

## 6.1 The Four Fatal Property Mistakes (Detailed)

### Mistake 1: Wrong Property Logic

```cvl
// ❌ WRONG: Logic error
rule deposit_check(uint256 amount) {
    uint256 before = balance[user];
    deposit(amount);
    uint256 after = balance[user];
    
    // BUG: Should be >=, not >
    assert after > before + amount;  // Too strong!
}

// ✅ CORRECT
rule deposit_check(uint256 amount) {
    uint256 before = balance[user];
    deposit(amount);
    uint256 after = balance[user];
    
    assert after == before + amount;  // Exact expectation
}
```

### Mistake 2: Partial Coverage

```cvl
// ❌ PARTIAL: Only checks one scenario
rule withdraw_notEmpty(uint256 amount) {
    require balance[user] > 0;  // Only tests non-zero balance
    ...
}

// ✅ COMPLETE: Covers all scenarios
rule withdraw_correctness(uint256 amount) {
    uint256 bal = balance[user];
    
    if (bal >= amount) {
        // Should succeed
        withdraw@withrevert(amount);
        assert !lastReverted;
    } else {
        // Should revert
        withdraw@withrevert(amount);
        assert lastReverted;
    }
}
```

### Mistake 3: Property Duplication

```cvl
// Property A3 and B5 are duplicates!
invariant A3_solvency()
    nativeBalances[this] >= totalOwed()

invariant B5_funds_sufficient()
    nativeBalances[this] >= totalOwed()  // Same as A3!
```

**Solution:** Maintain property ID tracking in `candidate_properties.md`

### Mistake 4: Mimicking Implementation

See Section 1.1 for detailed anti-pattern.

## 6.2 Spec Readability Pitfalls

### Anti-Pattern: Complex Expressions

```cvl
// ❌ BAD: Hard to debug
assert (balanceOf(a) + balanceOf(b)) * price >= 
       (debt[a] + debt[b]) * interestRate + 
       (fee[a] + fee[b]) * feeMultiplier;

// ✅ GOOD: Broken down
uint256 totalBal = balanceOf(a) + balanceOf(b);
uint256 totalDebt = debt[a] + debt[b];
uint256 totalFee = fee[a] + fee[b];

uint256 assets = totalBal * price;
uint256 liabilities = totalDebt * interestRate + totalFee * feeMultiplier;

assert assets >= liabilities, "Assets must cover liabilities";
```

## 6.3 Preserved Block Pitfalls

### Anti-Pattern: Missing Invariant Dependencies

```cvl
// ❌ WRONG: Missing dependency
invariant levelB_property()
    condition_that_depends_on_levelA
{
    preserved {
        // MISSING: requireInvariant levelA_property();
    }
}

// ✅ CORRECT: Include dependency
invariant levelB_property()
    condition_that_depends_on_levelA
{
    preserved {
        requireInvariant levelA_property();  // Declare dependency!
    }
}
```

---

# INTEGRATION WITH YOUR FRAMEWORK

## Mapping to Framework Phases

| Framework Phase | Apply These Techniques |
|-----------------|------------------------|
| **Phase 2** | Sections 1.1, 1.2, 1.3 (Property Discovery, Prioritization) |
| **Phase 3.5** | Section 3.1, 3.2 (Invariant Patterns) |
| **Phase 7** | Section 3.3, 6.2 (Preserved Blocks, Readability) |
| **CE Debugging** | Section 2 (CE Investigation Process) |
| **Harness Design** | Section 4 (Harness Best Practices) |
| **Loop Handling** | Section 5 (Loop Configuration) |

---

## Quick Checklist: Before Running Prover

```
□ Properties written in plain English first?
□ Properties don't mimic implementation?
□ Properties prioritized by impact?
□ Complex expressions broken down?
□ Preserved blocks include dependencies?
□ Loop_iter set appropriately?
□ Harnesses expose needed internal state?
□ Property IDs tracked to avoid duplication?
```

---

> **"Coming up with meaningful properties is the most challenging part of the work."**  
> — Certora Tutorial Lesson 06
