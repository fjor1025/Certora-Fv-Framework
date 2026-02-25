# ADVANCED CLI OPTIONS & PERFORMANCE OPTIMIZATION

> **Version:** 1.2 (Framework v3.2 — Optimization Pressure + Temporal Depth + Design Hostility)  
> **Purpose:** Advanced CLI flags, timeout mitigation, performance optimization, and offensive verification strategies  
> **Audience:** Users experiencing timeouts, complex projects, or seeking advanced debugging  
> **Updated:** February 16, 2026 (based on Certora Documentation latest updates)

---

## TABLE OF CONTENTS

1. [Performance Optimization](#1-performance-optimization)
2. [Advanced Debugging](#2-advanced-debugging)
3. [Loop & Array Handling](#3-loop--array-handling)
4. [Multi-Version Projects](#4-multi-version-projects)
5. [Project-Level Operations](#5-project-level-operations)
6. [Harness Patterns](#6-harness-patterns)
7. [Practical Tips & Tricks](#7-practical-tips--tricks)
8. [Quick Command Reference](#8-quick-command-reference)
9. [Offensive Verification CLI Strategies (v3.0)](#9-offensive-verification-cli-strategies-v30)

---

# 1. PERFORMANCE OPTIMIZATION

## 1.1 Split Rules Strategy

**Flag:** `--split_rules <rule_pattern>`

**What it does:**
Runs computationally expensive rules in separate dedicated Prover jobs with more resources.

**When to use:**
- Some rules take much longer than others
- Rules timeout when run together
- Want to maximize resources for heavy invariants

**How it works:**
1. Creates separate job for each matching rule
2. Creates one additional job for all non-matching rules
3. Returns dashboard link showing all job statuses
4. Each split rule gets full resource allocation

**Example:**

_Command line:_
```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --split_rules "solvency_*" "totalSupply_*"
```

_Configuration file:_
```json
{
  "files": ["Bank.sol"],
  "verify": "Bank:Bank.spec",
  "split_rules": ["solvency_*", "totalSupply_*"]
}
```

**Real-world example from dual-governance:**
```bash
# Heavy rules that benefit from splitting
certoraRun certora/confs/Escrow.conf \
    --split_rules "solvency_ETH" "claimNextBatch_*"
```

**Pro Tips:**
- Split rules that take >10 minutes
- Use wildcards `*` to match rule patterns
- Monitor dashboard to identify slow rules
- Start with 2-3 split rules, not all rules

---

## 1.2 Multi-Assert Check Mode

**Flag:** `--multi_assert_check`

**What it does:**
Checks each `assert` statement separately instead of all together.

**How it works:**
For a rule with assertions `a1` and `a2`:
- Creates sub-rule `R1`: proves `a1`, removes `a2`
- Creates sub-rule `R2`: assumes `a1`, proves `a2`
- Rule passes only if all sub-rules pass

**When to use:**
1. **Timeout mitigation:** Checking assertions separately can avoid timeout
2. **Multiple counterexamples:** Get different CEs for each assertion

**Example:**

```cvl
rule withdraw_correctness(uint256 amount) {
    env e;
    
    uint256 balBefore = balanceOf(e.msg.sender);
    uint256 totalBefore = totalSupply();
    
    withdraw(e, amount);
    
    uint256 balAfter = balanceOf(e.msg.sender);
    uint256 totalAfter = totalSupply();
    
    // With --multi_assert_check, these are checked separately
    assert balAfter == balBefore - amount, "Balance decreased correctly";
    assert totalAfter == totalBefore - amount, "Total supply decreased correctly";
    assert balAfter <= totalAfter, "Balance <= total supply";
}
```

_Run command:_
```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --rule withdraw_correctness \
    --multi_assert_check
```

**When NOT to use:**
- Simple rules with 1-2 assertions
- When assertions are logically dependent (later assertions rely on earlier ones being true)
- Rules that already run quickly

**Performance note:**
Generates more sub-rules, which can increase total run time. Use strategically.

---

## 1.3 Control Flow Splitting Options

**Advanced timeout mitigation through control flow splitting.**

### Background: What is Control Flow Splitting?

The Prover internally splits program paths to verify them separately:
```cvl
rule example {
    if (owner == spender) {
        assert balance_after == balance_before;
    } else {
        assert balance_after == balance_before + amount;
    }
}
```

Internally becomes:
- Split 1: `require owner == spender; assert balance_after == balance_before;`
- Split 2: `require owner != spender; assert balance_after == balance_before + amount;`

### Key Flags

| Flag | Purpose | Default | Use Case |
|------|---------|---------|----------|
| `--prover_args '-depth N'` | Max split depth | 10 | Increase for complex paths |
| `--prover_args '-mediumTimeout N'` | Timeout for non-leaf splits (seconds) | 10 | Increase for large code |
| `--prover_args '-smt_initialSplitDepth N'` | Eager split depth | 0 | Skip checks, split immediately |
| `--prover_args '-splitParallel true'` | Parallel split solving | false | Speed up with many splits |
| `--prover_args '-dontStopAtFirstSplitTimeout true'` | Continue after timeout | false | Looking for SAT (CE exists) |

### Strategy 1: Lazy Splitting (Many Medium Splits)

**When:** Many subproblems of medium difficulty

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-mediumTimeout 30 -depth 5'
```

**Effect:** Give solver more time before splitting again

---

### Strategy 2: Eager Splitting (Very Large Code)

**When:** Source code is very large, shallow splits too big for solvers

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-smt_initialSplitDepth 5 -depth 15'
```

**Effect:** Split immediately to depth 5 without checking, then continue normally

---

### Strategy 3: Parallel Splitting (Many Splits)

**When:** Many independent subproblems to solve

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-splitParallel true'
```

**Effect:** Solve splits concurrently instead of sequentially

---

### Strategy 4: Looking for Counterexamples

**When:** Expect rule to fail, want to find CE despite timeouts

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-dontStopAtFirstSplitTimeout true -depth 15 -mediumTimeout 5' \
    --smt_timeout 10
```

**Effect:** Continue searching even if some splits timeout (useful for finding bugs)

---

### Practical Decision Tree for Control Flow Splitting

```
Is your rule timing out?
│
├─► YES → Check run statistics (path count, nonlinear ops, memory complexity)
│         │
│         ├─► High Path Count?
│         │   │
│         │   ├─► Very large code? → Eager splitting
│         │   │   certoraRun ... --prover_args '-smt_initialSplitDepth 5 -depth 15'
│         │   │
│         │   ├─► Many medium splits? → Lazy splitting
│         │   │   certoraRun ... --prover_args '-mediumTimeout 30 -depth 5'
│         │   │
│         │   └─► Many independent splits? → Parallel splitting
│         │       certoraRun ... --prover_args '-splitParallel true'
│         │
│         ├─► High Nonlinear Arithmetic? → See Section 1.4
│         │
│         └─► High Memory Complexity? → Simplify summaries, reduce array bounds
│
└─► NO → Good! Consider split_rules for future-proofing heavy invariants
```

---

## 1.4 Timeout Mitigation Checklist

When you hit a timeout, follow this systematic approach:

### Step 1: Identify the Cause
```bash
# Run with verbose output to see statistics
certoraRun Bank.sol --verify Bank:Bank.spec --rule slow_rule
```

Check in the run report:
- **Path count:** Number of execution paths
- **Nonlinear operations:** Multiplications/divisions of variables
- **Memory/Storage complexity:** Number of reads/writes

### Step 2: Quick Wins

| Problem | Quick Fix | Command |
|---------|-----------|---------|
| Rule too broad | Run single rule | `--rule specific_rule` |
| Multiple slow rules | Split heavy rules | `--split_rules "pattern"` |
| Complex assertions | Split assertions | `--multi_assert_check` |
| Loop complexity | Adjust loop bounds | `--loop_iter N --optimistic_loop` |

### Step 3: Advanced Techniques

**A. Simplify Summaries**
```cvl
// Instead of DISPATCHER (path explosion)
function _.externalCall() external => DISPATCHER(true);

// Use AUTO or NONDET
function _.externalCall() external => AUTO;
```

**B. Modularization**
```cvl
// Break complex rule into smaller rules
rule transfer_security_suite {
    // Too complex!
    // Checks balance, allowance, total supply, events
}

// Better: Split into focused rules
rule transfer_updates_balance { ... }
rule transfer_respects_allowance { ... }
rule transfer_preserves_total_supply { ... }
```

**C. Control Flow Splitting** (see Section 1.3)

**D. Reduce Parametric Scope**
```bash
# Instead of all methods
certoraRun Bank.sol --verify Bank:Bank.spec

# Focus on specific methods
certoraRun Bank.sol --verify Bank:Bank.spec \
    --method "deposit(uint256)" --method "withdraw(uint256)"
```

---

# 2. ADVANCED DEBUGGING

## 2.1 Multi-Example Mode

**Flag:** `--multi_example`

**What it does:**
Shows multiple counterexamples (or witnesses) from different paths/reasons instead of just one.

**When to use:**
- Debugging complex rules with multiple ways to fail
- Want to see different edge cases
- Understanding full spectrum of violations

**Default behavior:**
- Prover returns ONE counterexample per failed rule
- Picks the first violation it finds

**With `--multi_example`:**
- Prover attempts to generate multiple examples
- Shows different control-flow paths
- Reveals different logical reasons for failure

**Example:**

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --rule deposit_increases_balance \
    --multi_example
```

**Real-world benefit:**
You might discover:
- CE 1: Violation due to overflow
- CE 2: Violation due to reentrancy
- CE 3: Violation due to external call failure

All from the SAME rule!

**Pro Tips:**
- Use with `--rule` to focus on one failing rule
- Review ALL examples to understand full attack surface
- Document each distinct CE pattern in your report

---

## 2.2 Independent Satisfy Mode

**Flag:** `--independent_satisfy`

**What it does:**
Checks each `satisfy` statement independently, ignoring other satisfy statements.

**Background:**
Normally, satisfy statements accumulate:
```cvl
rule example {
    bool b;
    satisfy b, "R1";        // Sub-rule 1: satisfy b
    satisfy !b, "R2";       // Sub-rule 2: satisfy b && !b (contradiction!)
}
```
Without `--independent_satisfy`, R2 fails (can't satisfy both `b` and `!b`).

**With `--independent_satisfy`:**
```cvl
// Sub-rule R1: satisfy b (ignores satisfy !b)
// Sub-rule R2: satisfy !b (ignores satisfy b)
```
Both succeed!

**When to use:**
- Demonstrating multiple independent scenarios
- Showing that different states are reachable
- Testing feasibility of different conditions

**Example:**

```cvl
rule state_machine_reachability {
    env e;
    
    // Can system reach Normal state?
    satisfy getCurrentState() == State.Normal, "Normal reachable";
    
    // Can system reach VetoSignalling state?
    satisfy getCurrentState() == State.VetoSignalling, "VetoSignalling reachable";
    
    // Can system reach VetoCooldown state?
    satisfy getCurrentState() == State.VetoCooldown, "VetoCooldown reachable";
    
    // Can system reach RageQuit state?
    satisfy getCurrentState() == State.RageQuit, "RageQuit reachable";
}
```

_Run with independent_satisfy:_
```bash
certoraRun DualGovernance.sol --verify DualGovernance:DualGovernance.spec \
    --rule state_machine_reachability \
    --independent_satisfy
```

**Result:** Each state checked independently, showing all are reachable.

---

## 2.3 Breaking Down Complex Expressions

**Best Practice from Tutorials:**

```cvl
// ❌ BAD: Hard to debug
assert balanceOf(user) + debt[user] <= totalSupply() + totalDebt();
```

**Problem:** In counterexample, you only see final boolean result (false)

```cvl
// ✅ GOOD: Each value visible in call trace
uint256 userTotal = balanceOf(user) + debt[user];
uint256 systemTotal = totalSupply() + totalDebt();
assert userTotal <= systemTotal, "User total exceeds system total";
```

**Benefits:**
1. Call trace shows `userTotal` and `systemTotal` values
2. Easy to spot where logic breaks
3. Better error messages
4. Faster debugging

**Apply to all complex rules:**
```cvl
rule withdraw_security {
    env e;
    uint256 amount;
    
    // Break down preconditions
    uint256 userBalance = balanceOf(e.msg.sender);
    uint256 userAllowance = allowance(e.msg.sender, e.msg.sender);
    bool hasBalance = userBalance >= amount;
    bool hasAllowance = userAllowance >= amount;
    require hasBalance && hasAllowance;
    
    // Break down state changes
    uint256 balBefore = userBalance;
    uint256 totalBefore = totalSupply();
    
    withdraw(e, amount);
    
    uint256 balAfter = balanceOf(e.msg.sender);
    uint256 totalAfter = totalSupply();
    
    // Break down assertions with descriptive messages
    uint256 expectedBal = balBefore - amount;
    assert balAfter == expectedBal, "Balance decreased by exact amount";
    
    uint256 expectedTotal = totalBefore - amount;
    assert totalAfter == expectedTotal, "Total supply decreased by amount";
}
```

---

# 3. LOOP & ARRAY HANDLING

> **From Certora Tutorial Lesson 11 & 12**

## 3.1 The Loop Problem

**Two fundamental issues:**
1. **Halting problem:** Loop might never terminate
2. **Complexity explosion:** Each iteration multiplies paths

**Certora's solution:** Loop unrolling (approximate the loop behavior)

## 3.2 Loop Flags

### `--loop_iter N`

**What it does:**
Unrolls loops up to `N` iterations.

**Default:** 1 iteration

**Example:**
```solidity
function multiTransfer(address[] calldata recipients, uint256[] calldata amounts) external {
    for (uint i = 0; i < recipients.length; i++) {
        transfer(recipients[i], amounts[i]);
    }
}
```

```bash
# Unroll loop 3 times (tests arrays of length 0, 1, 2, 3)
certoraRun Bank.sol --verify Bank:Bank.spec --loop_iter 3
```

**When to use:**
- Contract has loops over arrays
- Need to verify behavior for arrays of specific lengths
- Want to test loop logic correctness

**⚠️ Warning:** Higher N = exponentially more complexity

**Pro Tip:** Start with `--loop_iter 1`, increase only if needed

---

### `--optimistic_loop`

**What it does:**
Assumes loop doesn't change values that were checked before the loop.

**When to use:**
- Loop doesn't mutate state that affects the property
- Loop is side-effect free (e.g., view functions)
- Trying to avoid timeout from loop complexity

**Example:**

```solidity
function getTotalBalance(address[] calldata users) external view returns (uint256) {
    uint256 total = 0;
    for (uint i = 0; i < users.length; i++) {
        total += balanceOf(users[i]);  // Read-only, no mutations
    }
    return total;
}
```

```bash
certoraRun Bank.sol --verify Bank:Bank.spec --optimistic_loop
```

**⚠️ DANGER:** If loop DOES mutate state, results are unsound (may miss bugs)

**Safe usage pattern:**
```solidity
// SAFE for --optimistic_loop (read-only)
function sumBalances(address[] calldata users) external view returns (uint256) {
    uint256 sum = 0;
    for (uint i = 0; i < users.length; i++) {
        sum += balances[users[i]];  // Only reads
    }
    return sum;
}

// UNSAFE for --optimistic_loop (mutates state)
function multiTransfer(address[] calldata recipients, uint256[] calldata amounts) external {
    for (uint i = 0; i < recipients.length; i++) {
        balances[msg.sender] -= amounts[i];      // Mutates state!
        balances[recipients[i]] += amounts[i];   // Mutates state!
    }
}
```

---

## 3.3 Loop Handling Strategy

**From Tutorial Lesson 11:**

### Step 1: Run WITHOUT loop flags
```bash
certoraRun Bank.sol --verify Bank:Bank.spec
```
See what happens naturally.

### Step 2: Add `--loop_iter` for specific unrolling
```bash
certoraRun Bank.sol --verify Bank:Bank.spec --loop_iter 3
```
Test arrays of length 0-3.

### Step 3: Check if rule passes vacuously
```cvl
rule multiTransfer_preserves_funds {
    // ... rule logic ...
    
    assert totalFunds_after == totalFunds_before;
    
    // Add at end to ensure non-vacuous
    assert false, "Rule should not reach here";
}
```
If `assert false` passes, rule is vacuous!

### Step 4: Use `--optimistic_loop` ONLY if safe
```bash
# Only if loop is read-only
certoraRun Bank.sol --verify Bank:Bank.spec --optimistic_loop
```

### Step 5: Document loop assumptions
```cvl
/**
 * @title multiTransfer preserves total funds
 * 
 * Loop handling:
 * - --loop_iter 3 used (tests arrays up to length 3)
 * - NOT using --optimistic_loop (loop mutates state)
 * - Verified non-vacuous with terminal assert false
 */
rule multiTransfer_preserves_funds { ... }
```

---

## 3.4 Array Handling Patterns

> **From Tutorial Lesson 12**

### Pattern 1: Array Uniqueness

**Challenge:** Verify array elements are unique

**Bad approach:**
```cvl
// ❌ WRONG: Fails on revert paths
invariant uniqueArray(uint256 i, uint256 j)
    get(i) != get(j);
```

**Good approach:**
```cvl
// ✅ CORRECT: Use withrevert and filtered preserved
invariant uniqueArray(uint256 i, uint256 j)
    i != j => getWithDefaultValue(i) != getWithDefaultValue(j)
    filtered { f -> f.selector != sig:remove(uint256).selector }
    {
        preserved push(address addr) {
            require frequency(addr) == 0;  // Use helper invariant
        }
    }
```

### Pattern 2: Frequency Tracking

**Challenge:** Track how many times an element appears

**Solution:**
```cvl
ghost mapping(address => mathint) frequency;

hook Sstore array[INDEX uint256 i] address newValue 
    (address oldValue) STORAGE {
    
    frequency[oldValue] = frequency[oldValue] - 1;
    frequency[newValue] = frequency[newValue] + 1;
}

invariant frequencyValid(address addr)
    frequency[addr] >= 0;

invariant uniquenessFromFrequency(uint256 i, uint256 j)
    i != j => get(i) != get(j)
    {
        preserved {
            requireInvariant frequencyValid(get(i));
            requireInvariant frequencyValid(get(j));
        }
    }
```

### Pattern 3: Array Length Bounds

**Challenge:** Prevent excessive array lengths (gas/timeout)

```cvl
invariant maxArrayLength()
    arrayLength() <= 100
    {
        preserved push(address addr) {
            require arrayLength() < 100;
        }
    }
```

**Use in rules:**
```cvl
rule multiTransfer_sound {
    requireInvariant maxArrayLength();  // Bound length
    
    // ... rest of rule ...
}
```

---

# 4. MULTI-VERSION PROJECTS

> **Critical for real-world audits with mixed compiler versions**

## 4.1 Compiler Map

**Flag:** `--compiler_map` (or `--solc_map` in config)

**When to use:**
- Different files need different compiler versions
- Project uses Solidity + Vyper
- Legacy contracts alongside new contracts

**Example:**

_Command line:_
```bash
certoraRun Bank.sol Exchange.sol Token.vy \
    --verify Bank:Bank.spec \
    --compiler_map Bank.sol=solc4.25,Exchange.sol=solc6.7,Token.vy=vyper0.3.10
```

_Configuration file:_
```json
{
  "files": ["Bank.sol", "Exchange.sol", "Token.vy"],
  "verify": "Bank:Bank.spec",
  "solc_map": {
    "Bank.sol": "solc4.25",
    "Exchange.sol": "solc6.7",
    "Token.vy": "vyper0.3.10"
  }
}
```

**Pattern matching:**
```bash
# Wildcard for contract names
--compiler_map "Legacy*=solc4.25,Modern*=solc8.19"

# Path patterns
--compiler_map "src/legacy/**/*.sol=solc4.25,src/**/*.sol=solc8.19"
```

**Rules:**
1. All files must be mapped (no defaults)
2. Map is checked in order of appearance
3. First match wins (be specific first, general later)

---

## 4.2 Optimization Map

**Flag:** `--solc_optimize_map`

**When to use:**
Different contracts optimized differently (common in production).

**Example:**

```json
{
  "files": ["Bank.sol", "Exchange.sol"],
  "verify": "Bank:Bank.spec",
  "solc_optimize_map": {
    "Bank": "200",
    "Exchange": "1000000"
  }
}
```

**Real-world example:**
```bash
# Libraries: heavily optimized for gas
# Core contracts: lightly optimized for code size
certoraRun Core.sol Libraries.sol \
    --verify Core:Core.spec \
    --solc_optimize_map "Core=200,Libraries=1000000"
```

---

## 4.3 EVM Version Map

**Flag:** `--solc_evm_version_map`

**When to use:**
- Contracts target different EVM versions
- Testing upgrade compatibility
- L2 deployment (some use older EVM versions)

**Example:**

```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --solc_evm_version_map "Bank=cancun,Exchange=shanghai"
```

**Common EVM versions:**
- `cancun` (latest, 2024+)
- `shanghai` (2023)
- `paris` (The Merge, 2022)
- `london` (EIP-1559, 2021)
- `istanbul` (2019)

---

## 4.4 Via-IR Map

**Flag:** `--solc_via_ir_map`

**When to use:**
- Some contracts need IR pipeline (stack too deep issues)
- Other contracts fail with IR pipeline
- Mix compilation modes in same project

**Example:**

```json
{
  "files": ["Complex.sol", "Simple.sol"],
  "verify": "Complex:Complex.spec",
  "solc_via_ir_map": {
    "Complex": true,
    "Simple": false
  }
}
```

**Note:** If `--solc_via_ir` not set globally, all contracts default to false unless specified in map.

---

## 4.5 Complete Multi-Version Example

**Scenario:** Audit with legacy contracts + new contracts + Vyper token

```json
{
  "files": [
    "src/legacy/Bank.sol",
    "src/modern/Exchange.sol",
    "src/modern/Complex.sol",
    "lib/Token.vy"
  ],
  "verify": "Exchange:specs/Exchange.spec",
  
  "solc_map": {
    "src/legacy/**/*.sol": "solc4.25",
    "src/modern/**/*.sol": "solc8.19",
    "lib/**/*.vy": "vyper0.3.10"
  },
  
  "solc_optimize_map": {
    "Bank": "200",
    "Exchange": "200",
    "Complex": "200"
  },
  
  "solc_evm_version_map": {
    "src/legacy/**/*.sol": "istanbul",
    "src/modern/**/*.sol": "cancun"
  },
  
  "solc_via_ir_map": {
    "Complex": true
  }
}
```

---

# 5. PROJECT-LEVEL OPERATIONS

## 5.1 Project Sanity

**Flag:** `--project_sanity`

**What it does:**
Runs builtin sanity rule on ALL methods in ALL contracts in the project.

**How it finds files:**
- If in git repository: all `.sol` files in repo
- Otherwise: all `.sol` files in directory tree

**When to use:**
- Starting new verification project
- "Getting a feel" for project complexity
- Identifying hot spots for summarization
- Quick health check

**Example:**

```bash
certoraRun --project_sanity
```

**Output:** Report showing which methods:
- ✅ Pass sanity (can execute without reverting)
- ❌ Fail sanity (always revert or timeout)
- ⏱️ Timeout (too complex)

**Use results to:**
1. Identify methods that need summaries
2. Find overly complex methods
3. Prioritize verification efforts

**Note:** Automatically enables `--auto_dispatcher`

---

## 5.2 Foundry Integration

**Flag:** `--foundry`

**What it does:**
Runs Certora formal verification on ALL Foundry fuzz tests.

**How it works:**
1. Finds all files ending with `.t.sol`
2. Extracts fuzz test functions
3. Runs `verifyFoundryFuzzTestsNoRevert` builtin rule

**When to use:**
- Project already has Foundry tests
- Want formal verification on fuzz tests
- Bridge existing tests to formal verification

**Example:**

```bash
certoraRun --foundry
```

**What gets verified:**
```solidity
// In test/Bank.t.sol
contract BankTest is Test {
    // This gets formally verified!
    function testFuzz_deposit(uint256 amount) public {
        vm.assume(amount > 0 && amount < type(uint256).max);
        bank.deposit(amount);
        assertEq(bank.balanceOf(address(this)), amount);
    }
    
    // This too!
    function testFuzz_withdraw(uint256 amount) public {
        bank.deposit(amount);
        bank.withdraw(amount);
        assertEq(bank.balanceOf(address(this)), 0);
    }
}
```

**Pro Tips:**
- Use `vm.assume()` to constrain inputs
- Keep fuzz tests focused (one property per test)
- Avoid complex setup in fuzz tests

**Note:** Automatically enables `--auto_dispatcher`

---

## 5.3 Ignoring Solidity Warnings

**Flag:** `--ignore_solidity_warnings`

**What it does:**
Prevents Solidity compiler warnings from blocking verification.

**When to use:**
- Warning 6321: "Unnamed return variable can remain unassigned"
- Legacy code with known stylistic issues
- Non-critical warnings irrelevant to formal verification

**Example:**

```bash
certoraRun Token.sol --verify Token:Token.spec --ignore_solidity_warnings
```

**Common safe warnings to ignore:**
- Unnamed return variables (Solc 0.7.6+)
- Unused function parameters
- State variable shadowing (when intentional)

**⚠️ Warning:** Don't ignore warnings blindly. Review them first!

---

# 6. HARNESS PATTERNS

> **From Tutorial Lesson 15**

## 6.1 When to Use a Harness

**Use a harness when:**
- Contract has no public getters for internal state
- Need to expose storage for verification
- Want to simplify complex functions for testing
- Need helper functions for specifications

**Don't use a harness when:**
- Contract already has sufficient public API
- Would duplicate existing functionality
- Can use CVL ghosts/hooks instead

---

## 6.2 Harness Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Borda.sol";

/**
 * @title BordaHarness
 * @notice Harness for Borda contract formal verification
 * 
 * Exposes internal state and provides helper functions
 * for Certora specifications.
 */
contract BordaHarness is Borda {
    
    // ============ State Exposure ============
    
    /**
     * @notice Get voter registration status
     * @dev Exposes internal mapping
     */
    function getVoterRegistered(address voter) external view returns (bool) {
        return voters[voter].isRegistered;
    }
    
    /**
     * @notice Get voter blacklist status
     * @dev Exposes internal mapping
     */
    function getVoterBlacklisted(address voter) external view returns (bool) {
        return voters[voter].isBlacklisted;
    }
    
    /**
     * @notice Get voter attempts count
     * @dev Exposes internal mapping
     */
    function getVoterAttempts(address voter) external view returns (uint256) {
        return voters[voter].attempts;
    }
    
    /**
     * @notice Get contender points
     * @dev Exposes internal mapping
     */
    function getContenderPoints(uint256 contenderId) external view returns (uint256) {
        return contenders[contenderId].points;
    }
    
    // ============ Helper Functions ============
    
    /**
     * @notice Check if voter has voted
     * @dev Helper for specifications
     */
    function hasVoted(address voter) external view returns (bool) {
        return voters[voter].isRegistered && voters[voter].hasVoted;
    }
    
    /**
     * @notice Get total voters count
     * @dev Helper for monotonicity invariants
     */
    function getTotalVoters() external view returns (uint256) {
        return totalVoters;
    }
    
    /**
     * @notice Get total points distributed
     * @dev Helper for conservation invariants
     */
    function getTotalPoints() external view returns (uint256) {
        return totalPointsDistributed;
    }
}
```

---

## 6.3 Harness Best Practices

### ✅ DO:

1. **Keep it minimal**
   ```solidity
   // GOOD: Simple getter
   function getBalance(address user) external view returns (uint256) {
       return balances[user];
   }
   ```

2. **Mirror internal structure**
   ```solidity
   // If internal: mapping(address => uint256) balances;
   // Harness: expose as-is, don't transform
   function getBalance(address user) external view returns (uint256) {
       return balances[user];  // Direct access, no logic
   }
   ```

3. **Document purpose**
   ```solidity
   /**
    * @notice Exposes internal balance for solvency verification
    * @dev Used in invariant: sum(balances) <= totalSupply
    */
   function getBalance(address user) external view returns (uint256) {
       return balances[user];
   }
   ```

4. **Use clear naming**
   ```solidity
   // GOOD
   function getVoterRegistered(address voter) external view returns (bool);
   
   // BAD
   function x(address a) external view returns (bool);
   ```

### ❌ DON'T:

1. **Don't add complex logic**
   ```solidity
   // BAD: Complex computation
   function getAverageBalance() external view returns (uint256) {
       uint256 sum = 0;
       for (uint i = 0; i < users.length; i++) {
           sum += balances[users[i]];
       }
       return sum / users.length;
   }
   
   // GOOD: Expose raw data, compute in CVL
   function getBalance(address user) external view returns (uint256) {
       return balances[user];
   }
   ```

2. **Don't modify state**
   ```solidity
   // BAD: Mutates state
   function getAndResetBalance(address user) external returns (uint256) {
       uint256 bal = balances[user];
       balances[user] = 0;
       return bal;
   }
   
   // GOOD: Read-only
   function getBalance(address user) external view returns (uint256) {
       return balances[user];
   }
   ```

3. **Don't duplicate public functions**
   ```solidity
   // If contract already has: function balanceOf(address) external view returns (uint256)
   // DON'T add: function getBalance(address) external view returns (uint256)
   ```

---

## 6.4 Harness vs. CVL Ghosts

**When to use Harness:**
- Need to expose internal mappings
- Want to reuse in multiple specs
- Team prefers Solidity over CVL

**When to use CVL Ghosts:**
- Need to track aggregates (sum, count)
- Want to verify history (was value ever X?)
- Need mathematical variables (e.g., max value seen)

**Example comparison:**

_Harness approach:_
```solidity
// BankHarness.sol
function getBalance(address user) external view returns (uint256) {
    return balances[user];
}
```
```cvl
// Bank.spec
rule solvency {
    address user;
    uint256 balance = getBalance(user);
    assert balance <= totalSupply();
}
```

_Ghost approach:_
```cvl
// Bank.spec
ghost mathint sumBalances;

hook Sstore balances[KEY address user] uint256 newVal (uint256 oldVal) {
    sumBalances = sumBalances + newVal - oldVal;
}

invariant solvency()
    sumBalances <= to_mathint(totalSupply());
```

**Decision tree:**
```
Need to track SUM or COUNT?
│
├─► YES → Use CVL Ghost
│
└─► NO → Need simple value exposure?
         │
         ├─► YES → Use Harness
         │
         └─► NO → Maybe you don't need either!
```

---

# 7. PRACTICAL TIPS & TRICKS

## 7.1 Coverage Analysis

**Flag:** `--coverage_info <none|basic|advanced>`

**What it does:**
Analyzes coverage of Solidity code and CVL specs.

**Modes:**
- `none` (default): No coverage analysis
- `basic`: Which Solidity lines covered, which CVL assertions needed
- `advanced`: Deep analysis with visualization

**When to use:**
After rules pass, to find gaps in verification.

**Example:**

```bash
certoraRun Bank.sol --verify Bank:Bank.spec --coverage_info advanced
```

**Output shows:**
- Solidity lines covered by rules
- Solidity lines never reached
- CVL assertions that could be removed (always true)
- CVL requires that could be removed (always true)

---

## 7.2 Rule Sanity Checks

**Flag:** `--rule_sanity <none|basic|advanced>`

**What it does:**
Checks if rules can actually fail (non-vacuous).

**Modes:**
- `none`: No sanity check
- `basic`: Quick check for obvious vacuity
- `advanced`: Deep analysis

**When to use:**
- Rules pass too easily
- Suspicious quick verification
- Developing new rules

**Example:**

```bash
certoraRun Bank.sol --verify Bank:Bank.spec --rule_sanity basic
```

**Catches:**
```cvl
// Rule that can never fail (vacuous)
rule impossible {
    require false;  // Can never start!
    assert false;
}

// Rule with contradictory requires (vacuous)
rule contradiction {
    uint256 x;
    require x > 10;
    require x < 5;  // No x satisfies both!
    assert false;
}
```

---

## 7.3 Running Rules Individually

**Why:**
- Faster iteration during development
- Isolate failing rules
- Avoid timeout from running all rules

**Method 1: CLI Flag**
```bash
certoraRun Bank.sol --verify Bank:Bank.spec --rule specific_rule_name
```

**Method 2: Multiple Rules**
```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --rule "deposit_*" --rule "withdraw_*"
```

**Method 3: Exclude Rules**
```bash
certoraRun Bank.sol --verify Bank:Bank.spec \
    --exclude_rule "slow_*" --exclude_rule "experimental_*"
```

**Workflow:**
```bash
# Step 1: Run all rules to see which fail
certoraRun Bank.sol --verify Bank:Bank.spec

# Step 2: Focus on failing rule
certoraRun Bank.sol --verify Bank:Bank.spec --rule failing_rule

# Step 3: Debug and fix

# Step 4: Run all rules again to ensure no regressions
certoraRun Bank.sol --verify Bank:Bank.spec
```

---

## 7.4 Config File Best Practices

**Use .conf files for:**
- Projects with many options
- Reproducible runs
- CI/CD integration
- Team collaboration

**Template:**
```json
{
  "files": [
    "src/Bank.sol",
    "src/Token.sol"
  ],
  "verify": "Bank:specs/Bank.spec",
  "msg": "Bank verification - solvency properties",
  
  "solc": "solc8.19",
  "solc_optimize": "200",
  
  "rule": ["solvency_*", "deposit_*"],
  "exclude_rule": ["experimental_*"],
  
  "loop_iter": "3",
  "optimistic_loop": false,
  
  "multi_assert_check": true,
  
  "prover_args": [
    "-depth 10",
    "-mediumTimeout 20",
    "-splitParallel true"
  ]
}
```

**Organization strategy:**
```
certora/
├── confs/
│   ├── Bank-quick.conf          # Fast smoke test
│   ├── Bank-full.conf           # Complete verification
│   ├── Bank-solvency.conf       # Just solvency rules
│   └── Bank-debug.conf          # Single rule debugging
└── specs/
    ├── Bank.spec
    └── BankHelpers.spec
```

---

## 7.5 Message Documentation

**Flag:** `--msg "Description"`

**What it does:**
Adds description to run (shows in dashboard and email).

**Best practices:**

```bash
# BAD: Vague
certoraRun Bank.sol --verify Bank:Bank.spec --msg "test"

# GOOD: Specific
certoraRun Bank.sol --verify Bank:Bank.spec \
    --msg "Bank v2.0 - solvency invariants after overflow fix"

# BETTER: Includes ticket/PR reference
certoraRun Bank.sol --verify Bank:Bank.spec \
    --msg "PR #123 - Fix deposit reentrancy (issue CVT-456)"
```

**Helps with:**
- Tracking runs in dashboard
- Understanding run purpose later
- Correlating with code changes
- Team collaboration

---

# 8. QUICK COMMAND REFERENCE

## 8.1 Performance Optimization

```bash
# Split heavy rules
certoraRun Bank.sol --verify Bank:Bank.spec --split_rules "solvency_*"

# Multi-assert for timeout
certoraRun Bank.sol --verify Bank:Bank.spec --multi_assert_check

# Control flow splitting - eager
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-smt_initialSplitDepth 5 -depth 15'

# Control flow splitting - lazy
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-mediumTimeout 30 -depth 5'

# Parallel splits
certoraRun Bank.sol --verify Bank:Bank.spec \
    --prover_args '-splitParallel true'
```

---

## 8.2 Advanced Debugging

```bash
# Multiple counterexamples
certoraRun Bank.sol --verify Bank:Bank.spec --multi_example

# Independent satisfy statements
certoraRun Bank.sol --verify Bank:Bank.spec --independent_satisfy

# Rule sanity check
certoraRun Bank.sol --verify Bank:Bank.spec --rule_sanity basic

# Coverage analysis
certoraRun Bank.sol --verify Bank:Bank.spec --coverage_info advanced
```

---

## 8.3 Loop & Array Handling

```bash
# Loop unrolling
certoraRun Bank.sol --verify Bank:Bank.spec --loop_iter 3

# Optimistic loop (careful!)
certoraRun Bank.sol --verify Bank:Bank.spec --optimistic_loop

# Both together
certoraRun Bank.sol --verify Bank:Bank.spec --loop_iter 3 --optimistic_loop
```

---

## 8.4 Multi-Version Projects

```bash
# Compiler map
certoraRun Bank.sol Exchange.sol --verify Bank:Bank.spec \
    --compiler_map "Bank.sol=solc4.25,Exchange.sol=solc8.19"

# Optimization map
certoraRun Bank.sol --verify Bank:Bank.spec \
    --solc_optimize_map "Bank=200,Exchange=1000000"

# EVM version map
certoraRun Bank.sol --verify Bank:Bank.spec \
    --solc_evm_version_map "Bank=istanbul,Exchange=cancun"
```

---

## 8.5 Project-Level Operations

```bash
# Project sanity check
certoraRun --project_sanity

# Foundry integration
certoraRun --foundry

# Ignore warnings
certoraRun Token.sol --verify Token:Token.spec --ignore_solidity_warnings
```

---

## 8.6 Selective Running

```bash
# Single rule
certoraRun Bank.sol --verify Bank:Bank.spec --rule deposit_increases_balance

# Multiple rules by pattern
certoraRun Bank.sol --verify Bank:Bank.spec --rule "deposit_*" "withdraw_*"

# Exclude rules
certoraRun Bank.sol --verify Bank:Bank.spec --exclude_rule "slow_*"

# Specific methods for parametric rules
certoraRun Bank.sol --verify Bank:Bank.spec \
    --method "deposit(uint256)" --method "withdraw(uint256)"

# Name-only method filtering (v8.8.0+) — matches ALL overloads
certoraRun Bank.sol --verify Bank:Bank.spec \
    --method "deposit" --method "withdraw"

# Exclude methods by name (v8.8.0+) — name-only also supported
certoraRun Bank.sol --verify Bank:Bank.spec \
    --exclude_method "complexFunction"
```

---

## 8.7 Complete Real-World Example

```bash
# Production-ready command for complex project
certoraRun \
    src/Bank.sol \
    src/Token.sol \
    src/Oracle.sol \
    --verify Bank:specs/Bank.spec \
    --msg "Bank v3.2.1 - Full verification with timeout fixes" \
    \
    --solc solc8.19 \
    --solc_optimize 200 \
    --solc_via_ir \
    \
    --loop_iter 3 \
    --split_rules "solvency_ETH" "complex_invariant_*" \
    --multi_assert_check \
    \
    --prover_args '-depth 12 -mediumTimeout 25 -splitParallel true' \
    --smt_timeout 300 \
    \
    --rule_sanity basic \
    --coverage_info basic
```

---

# APPENDIX A: CLI FLAG DECISION TREE

```
Is your verification...

├─► TIMING OUT?
│   │
│   ├─► Multiple slow rules? → --split_rules <pattern>
│   ├─► Complex assertions? → --multi_assert_check
│   ├─► Many paths? → --prover_args '-smt_initialSplitDepth N -depth M'
│   ├─► Loops involved? → --loop_iter N (careful with --optimistic_loop)
│   └─► Still timing out? → Simplify summaries, reduce parametric scope
│
├─► DEBUGGING?
│   │
│   ├─► Need multiple CEs? → --multi_example
│   ├─► Multiple satisfy? → --independent_satisfy
│   ├─► Passing too easily? → --rule_sanity basic
│   ├─► Complex expressions? → Break down in CVL
│   └─► Need coverage info? → --coverage_info advanced
│
├─► MULTI-VERSION PROJECT?
│   │
│   ├─► Different compilers? → --compiler_map
│   ├─► Different optimization? → --solc_optimize_map
│   ├─► Different EVM versions? → --solc_evm_version_map
│   └─► IR pipeline issues? → --solc_via_ir_map
│
├─► PROJECT ASSESSMENT?
│   │
│   ├─► New project? → --project_sanity
│   ├─► Has Foundry tests? → --foundry
│   └─► Coverage gaps? → --coverage_info advanced
│
└─► RUNNING QUICKLY?
    │
    ├─► Focus on one rule → --rule <name>
    ├─► Exclude slow rules → --exclude_rule <pattern>
    ├─► Specific methods → --method <signature_or_name>
    │   (v8.8.0+: name-only matches all overloads)
    └─► Split heavy rules → --split_rules <pattern>
```

---

# APPENDIX B: RECOMMENDED .CONF FILES

## Quick Verification (Smoke Test)
```json
{
  "files": ["src/Contract.sol"],
  "verify": "Contract:specs/Contract.spec",
  "msg": "Quick smoke test",
  
  "rule": ["basic_*"],
  "exclude_rule": ["slow_*", "heavy_*"],
  
  "short_output": true,
  "rule_sanity": "basic"
}
```

## Full Verification
```json
{
  "files": ["src/Contract.sol"],
  "verify": "Contract:specs/Contract.spec",
  "msg": "Full verification suite",
  
  "solc": "solc8.19",
  "solc_optimize": "200",
  
  "split_rules": ["solvency_*", "invariant_*"],
  "multi_assert_check": true,
  
  "loop_iter": "3",
  
  "prover_args": [
    "-depth 12",
    "-mediumTimeout 20",
    "-splitParallel true"
  ],
  
  "rule_sanity": "basic",
  "coverage_info": "basic"
}
```

## Debug Single Rule
```json
{
  "files": ["src/Contract.sol"],
  "verify": "Contract:specs/Contract.spec",
  "msg": "Debug specific rule",
  
  "rule": ["failing_rule"],
  "multi_example": true,
  
  "prover_args": ["-dontStopAtFirstSplitTimeout true"]
}
```

---

---

# 9. OFFENSIVE VERIFICATION CLI STRATEGIES (v3.0)

> **New in Framework v3.0:** CLI patterns specifically for anti-invariant rules, multi-step attack proofs, and profit-threshold searches.

## 9.1 Running Anti-Invariant Rules

Anti-invariant rules (from `impact-spec-template.md`) intentionally search for profitable attack paths. They use `satisfy` instead of `assert`, so the Prover searches for a *witness* rather than a proof.

```bash
# Run a single anti-invariant rule
certoraRun src/Vault.sol --verify Vault:specs/offensive_vault.spec \
    --rule attacker_cannot_profit \
    --multi_assert_check \
    --smt_timeout 600 \
    --msg "Anti-invariant: searching for profitable attack"
```

**Key flags for offensive rules:**

| Flag | Purpose | Offensive Use Case |
|------|---------|--------------------|
| `--rule <name>` | Target a single anti-invariant rule | Isolate `attacker_cannot_profit` or `find_max_profit_threshold` |
| `--smt_timeout 600` | Allow longer solver time | Multi-step attacks need more exploration time |
| `--multi_assert_check` | Check each assert independently | Find partial attack paths even if full chain times out |
| `--prover_args '-depth 20'` | Increase search depth | Multi-step sequences need deeper unrolling |
| `--independent_satisfy true` | Check each `satisfy` independently | Find multiple distinct attack witnesses |

## 9.2 Multi-Step Attack Timeout Strategy

Multi-step attack rules (from `multi-step-attacks-template.md`) chain 3+ function calls. These are expensive:

```bash
# Strategy: increase timeout + parallel splitting
certoraRun src/Vault.sol --verify Vault:specs/multi_step_offensive.spec \
    --rule three_step_attack_sequence \
    --smt_timeout 900 \
    --prover_args '-depth 25 -splitParallel true -mediumTimeout 30' \
    --msg "Multi-step: 3-call attack sequence"
```

**Decision tree for offensive rule timeouts:**

```
Offensive rule timing out?
│
├─► Single-step anti-invariant → increase --smt_timeout to 600s
│
├─► Multi-step (3+ calls) → increase depth + splitParallel
│   └─► Still timing out? → Split rule into 2-step sub-proofs
│
└─► Profit threshold search → Use iterative binary search approach
    (see impact-spec-template.md: find_max_profit_threshold)
```

## 9.3 Iterative Profit Threshold Search

The `find_max_profit_threshold` pattern (from `impact-spec-template.md`) requires iterative runs with different thresholds:

```bash
# Step 1: Start with high threshold
certoraRun ... --rule find_max_profit_threshold \
    --prover_args '-threshold 1000000000000000000' \
    --msg "Threshold search: 1 ETH"

# Step 2: If SATISFIED → attacker can extract ≥ threshold → raise it
# Step 3: If VIOLATED → threshold is too high → lower it
# Step 4: Binary search to convergence
```

## 9.4 Offensive `.conf` Pattern

See `offensive-pipeline.md` for complete `.conf` templates. Quick reference:

```json
{
  "files": ["src/Vault.sol", "src/Token.sol"],
  "verify": "Vault:specs/offensive_vault.spec",
  "msg": "Offensive: anti-invariant + multi-step",
  "rule": ["attacker_cannot_profit", "three_step_attack_sequence"],
  "smt_timeout": "600",
  "multi_assert_check": true,
  "independent_satisfy": true,
  "prover_args": ["-depth 20"],
  "rule_sanity": "basic"
}
```

---

**End of Advanced CLI Reference**

> "The best performance optimization is the one you don't need because you designed your verification strategy correctly from the start."

For more information, see:
- certora-master-guide.md (Main workflow)
- cvl-language-deep-dive.md (CVL language reference)
- verification-playbooks.md (Worked examples)
- best-practices-from-certora.md (Tutorial techniques)
- quick-reference-v1.3.md (Quick lookup)
- impact-spec-template.md (Anti-invariant & profit threshold rules — NEW v3.0)
- multi-step-attacks-template.md (Multi-step attack sequences — NEW v3.0)
- offensive-pipeline.md (CI/CD pipeline & CE triage — NEW v3.0)
