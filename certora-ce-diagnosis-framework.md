# Certora Counterexample Diagnosis Framework

> **Version:** 2.3 (Red Team Hardening + Validation Evidence Gate)  
> **Purpose:** Determine whether a Certora counterexample represents a REAL BUG or a SPEC BUG  
> **Philosophy:** A counterexample is not evidence until execution AND causal closure are verified.  
> **Source:** Enhanced with Certora Tutorial Lesson 02 investigation workflow

---

## Table of Contents

1. [Objective](#objective)
2. [Input Requirements](#input-requirements)
3. [Tutorial-Based Investigation Workflow](#tutorial-based-investigation-workflow)
4. [Phase -1: Closure Verification (Mandatory Gate)](#phase--1-closure-verification-mandatory-gate)
5. [Phase A: Counterexample Classification](#phase-a-counterexample-classification)
6. [Phase B: Spec Repair](#phase-b-spec-repair-only-if-spec-bug)
7. [Pattern Reference](#pattern-reference)
8. [Quick Decision Tree](#quick-decision-tree)
9. [Non-Negotiable Rules](#non-negotiable-rules)

---

## Objective

Determine whether the Certora counterexample represents:

| Classification | Meaning | Action |
|----------------|---------|--------|
| **REAL BUG** | Reachable on-chain exploit under valid threat model | Report to development team |
| **SPEC BUG** | Violation of execution reality or causal closure due to incomplete modeling | Fix the spec |
| **OUT OF SCOPE** | Requires trusted role misbehavior within their powers | Document and exclude |
| **SILENT PASS** | Rule passes because revert paths were pruned, not because behavior is correct | Add `@withrevert` | 

⚠️ **A counterexample is not evidence until execution AND causal closure are verified.**

⚠️ **NEW v1.6 — A passing rule is not evidence of correctness if revert paths were silently pruned.**
> By default, the Certora Prover ignores all execution paths that revert. This means:
> - A rule that calls `f(e, args)` (without `@withrevert`) only proves the happy path
> - If the function SHOULD revert under certain conditions, those conditions are never tested
> - The rule passes — but the behavior is incompletely specified
>
> **Diagnosis:** If a rule passes "too easily" or you expected a counterexample but got none,
> check whether the function is called without `@withrevert`. Add `@withrevert` and re-run.
> If the rule now fails, the original pass was a **Silent Pass**.
>
> **Example:**
> ```cvl
> // PASSES — but only proves happy path (overflow is silently pruned)
> rule add_check() {
>     uint256 a; uint256 b;
>     uint256 c = add(a, b);       // Reverts silently pruned!
>     assert a + b == c;           // Only checked when add succeeds
> }
>
> // FAILS — correctly exposes that overflow path is unhandled
> rule add_check_complete() {
>     uint256 a; uint256 b;
>     uint256 c = add@withrevert(a, b);   // Revert paths included
>     assert a + b == c;                  // Now tested for ALL inputs
> }
> ```

---

## Input Requirements

Before starting diagnosis, gather:

- [ ] Full Certora call trace (including HAVOC annotations)
- [ ] Contract source code under verification
- [ ] Spec file(s) including methods block, ghosts, hooks
- [ ] Summaries used in the run
- [ ] Mutation path analysis from validation phase
- [ ] Invariant dependency chain documentation

---

## Tutorial-Based Investigation Workflow

> **Source:** Certora Tutorial Lesson 02 - Investigate Violations

### The 5-Step Investigation Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   STEP 1: Run Entire Spec First                                         │
│   ─────────────────────────────                                         │
│   certoraRun config.conf                                                 │
│                                                                          │
│   → See which rules fail                                                │
│   → Get overview of issues                                              │
│   → Identify patterns across violations                                 │
│                                                                          │
│   STEP 2: Focus on One Rule                                             │
│   ──────────────────────────                                            │
│   certoraRun config.conf --rule specific_rule_name                       │
│                                                                          │
│   → Saves run time                                                      │
│   → Cleaner output                                                      │
│   → Easier to analyze call trace                                        │
│                                                                          │
│   STEP 3: Analyze Call Trace                                            │
│   ───────────────────────────                                           │
│   • Check storage values before/after each function call                │
│   • Check arguments passed to functions                                 │
│   • Check return values from functions                                  │
│   • Follow execution path step-by-step                                  │
│   • Look for HAVOC annotations (unresolved calls)                       │
│                                                                          │
│   STEP 4: Identify Deviation                                            │
│   ───────────────────────────                                           │
│   • Where does execution differ from specification?                     │
│   • Is the difference due to incomplete spec?                           │
│   • Is the difference due to contract bug?                              │
│   • Is the property too strong (impossible to satisfy)?                 │
│   • Is the implementation wrong (security bug)?                         │
│                                                                          │
│   STEP 5: Fix and Verify                                                │
│   ─────────────────────────────                                         │
│   • Fix the issue (spec OR contract)                                    │
│   • Re-run to confirm fix                                               │
│   • Document the fix and reasoning                                      │
│   • Check if fix affects other rules                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Call Trace Analysis Tips

> **From Tutorial:** "Call traces contain information regarding the storage, arguments and return value."

#### What to Check in Storage

| Element | Before Call | After Call | Analysis |
|---------|-------------|------------|----------|
| **State Variables** | Initial values | Changed values | Did expected mutations occur? |
| **Balances** | Starting amounts | Ending amounts | Correct arithmetic? |
| **Mappings** | Key-value pairs | Updated pairs | Proper updates? |
| **Counters** | Old count | New count | Increments correct? |

#### What to Check in Arguments

```
Function: transfer(address to, uint256 amount)

Check:
✓ Is `to` address valid? (not zero, not this)
✓ Is `amount` within bounds? (not overflow)
✓ Does msg.sender have sufficient balance?
✓ Are preconditions from spec satisfied?
```

#### What to Check in Return Values

```
Function: getBalance(address user) returns (uint256)

Check:
✓ Return value matches storage state
✓ Return value within expected range
✓ Return value consistent with spec
```

### Breaking Down Complex Expressions

> **Best Practice from Tutorial:**
> "Try breaking complex expressions to achieve code readability and a more simplified call trace."

```cvl
// ❌ Complex (hard to debug)
assert balanceOf(user) + debt[user] <= totalSupply() + totalDebt();

// ✅ Broken down (easier call trace)
uint256 userTotal = balanceOf(user) + debt[user];
uint256 systemTotal = totalSupply() + totalDebt();
assert userTotal <= systemTotal, "User total exceeds system total";
```

**Benefits of breaking down:**
1. Each intermediate value visible in call trace
2. Easier to spot where logic breaks
3. Better error messages
4. Simpler debugging

### Bug Documentation Template

When you identify a bug, document it:

```markdown
### Bug Found: [Rule Name] Violation

**Date:** [YYYY-MM-DD]

**Rule:** `[rule_name]`

**File:** `Contract.sol:LineNumber`

**Type:** [REAL BUG / SPEC BUG]

**Issue:**
[One-sentence description of what's wrong]

**Call Trace Summary:**
```
Function called: functionName(args)
Storage before: variable = X
Storage after: variable = Y
Expected: variable = Z
```

**Root Cause:**
[Detailed explanation of why the bug occurred]

**Fix:**
```solidity
// OLD (buggy)
require(a > b);

// NEW (fixed)
require(b > a);
```

**Reasoning:**
The require checked `a > b`, when it should've checked `b > a`.
This allowed [exploit description].

**Impact:**
- [ ] HIGH: Loss of funds / DoS / Privilege escalation
- [ ] MEDIUM: Accounting inconsistency
- [ ] LOW: Single function misbehavior

**Related Properties:**
- Property ID1 (also affected)
- Property ID2 (dependency)
```

---

## Phase -1: Closure Verification (Mandatory Gate)

> **Before reading the CE, complete ALL closure checks.**  
> **If ANY check fails, classify immediately as SPEC BUG and skip to Phase B.**

### [ ] -1.1 Execution Surface Closure

| Check | Status | Notes |
|-------|--------|-------|
| All external calls summarized (dispatcher/CONSTANT/justified assume) | ☐ | |
| All external reads have deterministic summaries or are modeled | ☐ | |
| All NONDET usages are justified and don't affect safety-critical paths | ☐ | |
| No ghost mirrors individual storage owned by a real contract | ☐ | |
| All read/write pairs modeled (balanceOf↔transfer, etc.) | ☐ | |
| All contract-typed addresses bound or constrained | ☐ | |
| Custom summaries don't introduce unrealistic behavior | ☐ | |

**NONDET Acceptability Checklist:**

| NONDET Function | Modifies Verified Contract? | Affects Control Flow? | Triggers Callback? | Justified? |
|-----------------|-----------------------------|-----------------------|-------------------|------------|
| [function1] | ☐ No | ☐ No | ☐ No | ☐ Yes |
| [function2] | ☐ No | ☐ No | ☐ No | ☐ Yes |

🚨 **If ANY box fails → SPEC BUG (Model Incomplete) → Skip to Phase B**

---

### [ ] -1.2 Causal Closure

| Check | Status | Notes |
|-------|--------|-------|
| Every variable in violated assertion has all mutation paths enumerated | ☐ | |
| No variable can change without passing through modeled call/constructor | ☐ | |
| Constructor-written storage is constrained or modeled | ☐ | |
| All ghosts have Sstore hooks for EVERY mutation path | ☐ | |
| All ghosts have Sload hooks enforcing relationships | ☐ | |
| All invariants have concrete establishment traces | ☐ | |

🚨 **If ANY box fails → SPEC BUG (Causal Incompleteness) → Skip to Phase B**

---

### [ ] -1.3 Bounded State

| Check | Status | Notes |
|-------|--------|-------|
| Array lengths in CE are realistic (< 1000) | ☐ | |
| Timestamps in CE are realistic (<= max_uint40, > 0) | ☐ | |
| Loop iterations don't exceed preserved block bounds | ☐ | |
| Counter/ID values are within realistic ranges | ☐ | |
| No uint256 values near max that would wrap | ☐ | |

🚨 **If ANY box fails → SPEC BUG (Missing Bounds) → Skip to Phase B**

---

### [ ] -1.4 Invariant Dependencies

| Check | Status | Notes |
|-------|--------|-------|
| All `requireInvariant` calls reference proven invariants | ☐ | |
| Invariant dependency chain has no cycles | ☐ | |
| Every invariant has `@dev Level: N` annotation | ☐ | ← NEW v1.9 |
| `requireInvariant` only references LOWER-level invariants | ☐ | ← NEW v1.9 |
| Invariant dependency DAG documented in `causal_validation.md` | ☐ | ← NEW v1.9 |
| Base invariants (Level 1, no dependencies) proven in isolation first (`--rule`) | ☐ | |
| Higher-level invariants proven only after their dependencies pass | ☐ | ← NEW v1.9 |
| Preserved blocks load all necessary dependencies | ☐ | |

🚨 **If ANY box fails → SPEC BUG (Dependency Violation) → Skip to Phase B**

> **Cycle Detection Protocol (NEW v1.9):**
> If you cannot assign a unique level N to every invariant such that all
> `requireInvariant` calls point to strictly lower levels, you have a
> circular dependency. This creates a logical loop where both invariants
> are assumed true without proof. **STOP and refactor the invariant chain.**

---

### Phase -1 Verdict

| All Checks Passed? | Action |
|--------------------|--------|
| ✅ Yes | Proceed to Phase A |
| ❌ No | Classify as SPEC BUG, proceed to Phase B |

---

## Phase A: Counterexample Classification

> **Only enter Phase A if ALL Phase -1 checks passed.**

### [ ] A.1 Record the Counterexample (OBSERVATION ONLY)

Do **not** reason yet. Fill in this template:

| Element | Value |
|---------|-------|
| **Violated property** | |
| **Property type** | ☐ Invariant ☐ Rule |
| **Functions called (ordered)** | 1. <br> 2. <br> 3. |
| **Caller addresses** | |
| **msg.value per call** | |
| **External calls made** | |
| **HAVOC points (if any)** | |
| **Storage keys changed** | |
| **Storage values: before** | |
| **Storage values: after** | |
| **Ghost values: before** | |
| **Ghost values: after** | |
| **block.timestamp used** | |
| **block.number used** | |
| **Revert occurred?** | ☐ Yes (where?) ☐ No |

🚫 **No judgments. No fixes. Just record.**

---

### [ ] A.2 HAVOC Location Identification

In Certora's call trace, check for HAVOC indicators:

| Location in Trace | What to Look For | Found? |
|-------------------|------------------|--------|
| "Ghosts State" section | `havoc` annotation on any ghost | ☐ |
| "External Call" entries | "returns HAVOC" or "NONDET" | ☐ |
| "Storage" section | Value changes with no modeled write | ☐ |
| Between-call state | Ghost differs from storage unexpectedly | ☐ |

**HAVOC Indicators:**

| Indicator | Meaning |
|-----------|---------|
| `value: *` | Unconstrained (HAVOC) |
| `havoc` annotation | Explicit HAVOC |
| Unexplained value change | Implicit HAVOC from external call |
| Ghost ≠ storage after external call | Ghost desync from HAVOC |

🚨 **Any HAVOC in the path to violation → SPEC BUG (should have been caught in -1.1)**

---

### [ ] A.3 Filtered Function Check

If the violated property is an invariant or parametric rule:

| Question | Answer |
|----------|--------|
| Was the CE function excluded by a `filtered` clause? | ☐ Yes ☐ No |
| If yes, is the exclusion justified? | ☐ Yes ☐ No |
| Should this function be able to affect the property? | ☐ Yes ☐ No |

🚨 **If CE function was incorrectly filtered → SPEC BUG (Incorrect Filtering)**

---

### [ ] A.4 Ownership of Truth Check

For every value used in the CE:

| Value | Owner Contract | Modeled As | Mutation Paths Enumerated? |
|-------|----------------|------------|---------------------------|
| | | | ☐ |
| | | | ☐ |
| | | | ☐ |

🚨 If any value has:
- No owner → **SPEC BUG**
- Multiple owners → **SPEC BUG**  
- Ghost + storage duplication → **SPEC BUG**

---

### [ ] A.5 Ghost Causality Check

For every ghost involved in the CE:

| Ghost | Depends on Storage | All Mutation Paths Have Hooks? | Synchronized in CE? |
|-------|-------------------|-------------------------------|---------------------|
| | | ☐ | ☐ |
| | | ☐ | ☐ |

🚨 If any ghost:
- Has missing hooks → **SPEC BUG (Ghost Desynchronization)**
- Shows different value than expected from storage → **SPEC BUG**

---

### [ ] A.6 Conservation Laws Check

Verify physical laws of the system:

| Law | Holds in CE? | If Violated, Depends on HAVOC/Causal Break? |
|-----|--------------|---------------------------------------------|
| No asset appears ex nihilo | ☐ | ☐ |
| Supply changes match mint/burn only | ☐ | ☐ |
| No balance goes negative | ☐ | ☐ |
| No balance exceeds total supply | ☐ | ☐ |
| External return values have backing storage | ☐ | ☐ |
| Token amounts are conserved across transfers | ☐ | ☐ |

🚨 Conservation violation that depends on HAVOC/causal break → **SPEC BUG**  
🚨 Conservation violation despite full closure → **REAL BUG**

---

### [ ] A.7 Multi-Contract Trace Analysis

If CE involves multiple contracts:

| Contract Address | Role in CE | Modeled As | Behavior Matches Summary? |
|------------------|-----------|------------|--------------------------|
| | | | ☐ |
| | | | ☐ |

| Check | Status |
|-------|--------|
| Each contract's behavior matches its summary | ☐ |
| DISPATCHER routing went to expected implementation | ☐ |
| No contract received unexpected callbacks | ☐ |

🚨 **Behavior mismatch → SPEC BUG (Summary Inconsistency)**

---

### [ ] A.8 The Attacker's Question

> **"Could an EVM adversary force this exact trace using only modeled execution paths?"**

**The attacker MAY use:**
- ✅ Valid calldata
- ✅ Legitimate call ordering
- ✅ Real reentrancy (if modeled)
- ✅ External contracts behaving per their summaries
- ✅ Publicly callable functions
- ✅ Flash loans (if in scope)

**The attacker may NOT rely on:**
- ❌ Unestablished invariants
- ❌ Symbolic initial states without establishment trace
- ❌ Constructor-injected state unreachable post-deployment
- ❌ HAVOC or NONDET behavior
- ❌ State changes without modeled mutation paths
- ❌ Values outside realistic bounds
- ❌ Privileged role access (unless that's the threat model)

**You may NOT assume:**
- ❌ Honest token behavior
- ❌ Stable prices
- ❌ "Expected behavior" not enforced by code
- ❌ Invariants that aren't causally established

**Attacker Feasibility Assessment:**

| Step in CE | Attacker Can Achieve? | How? |
|------------|----------------------|------|
| 1. | ☐ Yes ☐ No | |
| 2. | ☐ Yes ☐ No | |
| 3. | ☐ Yes ☐ No | |

---

### [ ] A.9 Trusted Role Boundary Check

If the trace requires a privileged actor:

| Question | Answer |
|----------|--------|
| What role is involved? | |
| What power does code grant this role? | |
| Does CE require role to exceed code-granted power? | ☐ Yes ☐ No |
| Does CE require role to use power maliciously? | ☐ Yes ☐ No |
| Is role honesty assumed but not enforced? | ☐ Yes ☐ No |

**Classification:**

| Condition | Classification |
|-----------|----------------|
| Role power exceeds code | **REAL BUG** |
| Role uses code power to break safety | **REAL BUG** |
| Role acts within power, no invariant violated | **OUT OF SCOPE** |
| Role behavior assumed honest but not enforced | **SPEC BUG (Threat Model Leak)** |

🔴 **Rule:** If a role is assumed honest but not enforced by code, the spec is wrong, not the protocol.

---

### [ ] A.10 Verdict Gate

| Condition | Verdict |
|-----------|---------|
| Execution closure violated (Phase -1.1) | **SPEC BUG** |
| Causal closure violated (Phase -1.2) | **SPEC BUG** |
| Bounds violated (Phase -1.3) | **SPEC BUG** |
| Dependencies violated (Phase -1.4) | **SPEC BUG** |
| HAVOC in critical path (A.2) | **SPEC BUG** |
| Incorrect filtering (A.3) | **SPEC BUG** |
| Ownership unclear (A.4) | **SPEC BUG** |
| Ghost desync (A.5) | **SPEC BUG** |
| Conservation violation via causal break (A.6) | **SPEC BUG** |
| Summary inconsistency (A.7) | **SPEC BUG** |
| Attacker CAN achieve via modeled paths (A.8) | **REAL BUG** |
| Requires trusted role within their power | **OUT OF SCOPE** |
| Attacker CANNOT achieve, no closure issues | **SPEC BUG** (investigate further) |

**Final Verdict:** ☐ REAL BUG ☐ SPEC BUG ☐ OUT OF SCOPE

**Justification:**
```
[Write brief explanation citing specific checks that led to verdict]
```

---

## Phase B: Spec Repair (Only If SPEC BUG)

### [ ] B.1 Identify the Broken Law

> **"Which execution or causal law did I fail to encode?"**

Pick exactly one:

| Category | Specific Law | Selected? |
|----------|--------------|-----------|
| Execution | Missing external summary | ☐ |
| Execution | Missing write for read | ☐ |
| Execution | NONDET on safety-critical function | ☐ |
| Execution | Unconstrained contract address | ☐ |
| Execution | Summary inconsistent with reality | ☐ |
| Causal | Unowned truth | ☐ |
| Causal | Ghost/storage duplication | ☐ |
| Causal | Missing mutation path | ☐ |
| Causal | Missing constructor causality | ☐ |
| Causal | Ghost desynchronization | ☐ |
| Causal | Invariant not established | ☐ |
| Bounds | Missing array bound | ☐ |
| Bounds | Missing timestamp bound | ☐ |
| Bounds | Missing loop bound | ☐ |
| Dependency | Missing requireInvariant | ☐ |
| Dependency | Cyclic dependency | ☐ |
| Filtering | Incorrect filtered exclusion | ☐ |

🚫 **If you cannot name the law → STOP and re-evaluate Phase -1**

---

### [ ] B.2 Choose the Correct Repair Tool

| Missing Law | CVL Fix |
|-------------|---------|
| Missing external summary | Add to methods block: `function _.func() external => DISPATCHER(true);` |
| Missing determinism | Add to methods block: `function _.func() external => CONSTANT;` |
| NONDET on critical function | Replace with DISPATCHER or custom summary |
| Unconstrained address | Add `using Contract as alias;` and bind in rules |
| Summary inconsistency | Revise summary to match actual behavior |
| Unowned truth | Identify owner, remove ghost or add proper hooks |
| Ghost/storage duplication | Remove ghost, use storage directly |
| Missing mutation path | Add function to mutation path enumeration |
| Constructor causality | Add `init_state axiom` or model constructor |
| Ghost desynchronization | Add Sstore hook for missing mutation path |
| Invariant not established | Add `requireInvariant` in preserved block |
| Missing array bound | Add `require array.length < 1000;` in validState() |
| Missing timestamp bound | Add `require e.block.timestamp <= max_uint40;` in validEnv() |
| Missing loop bound | Add `require array.length <= 5;` in specific preserved block |
| Missing requireInvariant | Add `requireInvariant dependentInvariant();` in preserved |
| Cyclic dependency | Refactor invariants to break cycle |
| Incorrect filtering | Adjust `filtered { f -> ... }` clause |

🚫 **Never fix by tightening `require` unless it encodes EVM reality**

**Self-Check:** Is this fix enforced by Solidity or EVM physics?
- ☐ Yes → Valid fix
- ☐ No → Invalid fix, find another approach

---

### [ ] B.3 Implement the Fix

**Fix Location:**
- ☐ Methods block
- ☐ Ghost declaration
- ☐ Hook
- ☐ validState() function
- ☐ validEnv() function  
- ☐ Invariant preserved block
- ☐ Rule preconditions
- ☐ filtered clause

**Code Change:**
```cvl
// Before:


// After:

```

---

### [ ] B.4 Constraint Validity Test

After implementing fix:

1. **Remove the fix temporarily**
2. **Re-run the prover**
3. **Evaluate result:**

| Result | Interpretation |
|--------|----------------|
| CE reappears | ✅ Fix provides real protection |
| CE disappears even without fix | ❌ Bug was required away, fix is wrong |
| Different CE appears | ⚠️ May have revealed another issue |

🔴 **Rule:** `assume` may only restore execution reality or causal closure, never remove behaviors.

---

### [ ] B.5 Vacuity & Reachability Check

**Option A: Use Certora's built-in check**
```bash
certoraRun ... --rule_sanity basic
```

**Option B: Write a sanity rule**
```cvl
rule myRule_sanity(env e, ...) {
    // Same setup as myRule
    
    // Verify at least one satisfying trace exists
    satisfy [condition_that_should_be_reachable];
}
```

**Option C: Manual verification**

| Check | Status |
|-------|--------|
| Rule has at least one realistic satisfying trace | ☐ |
| No trace requires impossible balances | ☐ |
| No trace requires unmodeled calls | ☐ |
| No trace requires infinite approvals | ☐ |
| No trace requires adversarial oracle behaving honestly | ☐ |
| No trace requires invariants without establishment | ☐ |
| No trace requires state changes without mutation paths | ☐ |

🚨 **If no satisfying trace → SPEC BUG (Vacuous Rule)**

---

### [ ] B.6 Re-run Verification

After fix is validated:

1. Run full verification
2. Check that:
   - [ ] Original CE no longer appears
   - [ ] No new SPEC BUG CEs introduced
   - [ ] Rule sanity passes
   - [ ] No vacuity warnings

---

## Pattern Reference

### Causal Failure Patterns

| Pattern | Symptom | Root Cause | Fix |
|---------|---------|------------|-----|
| Magic State Change | Storage changes without any call | Unmodeled mutation path or constructor | Enumerate mutation paths, model constructor |
| Ghost Drift | Ghost value diverges from storage | Missing ghost update on some path | Add Sstore hook for missing path |
| Invariant Bootstrap | Invariant holds at t=0, breaks after first call | Invariant never established through modeled path | Add establishment trace |
| Constructor HAVOC | Initial state inconsistent with deployment | Unconstrained constructor writes | Model constructor or add init_state axiom |
| Callback Havoc | State changes during external call | Unmodeled reentrancy | Model callback or add reentrancy guard |
| Oracle Manipulation | Price/rate changes unexpectedly | Oracle not constrained | Add oracle trust assumption with justification |

### HAVOC Source Identification

| HAVOC Source | How to Fix |
|--------------|------------|
| Unresolved external call | Add DISPATCHER or summary |
| Library call returning bytes | Add custom summary returning empty bytes |
| Delegate call | Model delegate target |
| Create/Create2 | Model factory deployment |
| Low-level call | Add custom summary |

### Common Bounds

| Element | Recommended Bound | Justification |
|---------|-------------------|---------------|
| Array length | `< 1000` | Gas limits |
| Timestamp | `<= max_uint40` | Year ~36812 |
| Token amounts | `< 2^128` | Realistic supply |
| Loop iterations (preserved) | `<= 5` | Prover performance |
| ID/Counter | `< 100000000` | Realistic usage |

---

## Quick Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│                 COUNTEREXAMPLE FOUND                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│           PHASE -1: CLOSURE VERIFICATION                    │
│                                                             │
│  -1.1 Execution surface closed?                             │
│  -1.2 Causal closure satisfied?                             │
│  -1.3 Bounds realistic?                                     │
│  -1.4 Dependencies satisfied?                               │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
        ANY FAILED                      ALL PASSED
              │                               │
              ▼                               ▼
┌─────────────────────┐         ┌─────────────────────────────┐
│     SPEC BUG        │         │      PHASE A: CLASSIFY      │
│  → Go to Phase B    │         └─────────────────────────────┘
└─────────────────────┘                       │
                                              ▼
                              ┌───────────────────────────────┐
                              │  Can attacker achieve this    │
                              │  via modeled paths only?      │
                              └───────────────────────────────┘
                                              │
                        ┌─────────────────────┼─────────────────────┐
                        │                     │                     │
                        ▼                     ▼                     ▼
                       YES                UNCLEAR                   NO
                        │                     │                     │
                        ▼                     ▼                     ▼
              ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
              │   REAL BUG      │   │ Check trusted   │   │   SPEC BUG      │
              │ Report to devs  │   │ role boundaries │   │ → Go to Phase B │
              └─────────────────┘   └─────────────────┘   └─────────────────┘
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                                    ▼                   ▼
                           Role exceeded         Role within power
                              power                     │
                                │                       ▼
                                ▼               ┌───────────────┐
                         ┌──────────────┐       │ OUT OF SCOPE  │
                         │  REAL BUG    │       └───────────────┘
                         └──────────────┘
```

---

## Non-Negotiable Rules

### The Final Rule

> **Never change the spec to kill a counterexample unless you can name the exact execution or causal law that was missing.**

### Corollaries

1. **HAVOC is always a spec bug** — it means reality is unconstrained
2. **Causal breaks are always spec bugs** — they mean your spec allows impossible transitions
3. **Real bugs only exist in causally closed universes** — otherwise you're fighting ghosts
4. **Never fix by adding safety constraints** — fix by modeling reality more accurately
5. **Never `require` away an exploit** — find the missing model
6. **Mark every invariant with its source of truth** — traceability is non-negotiable

### The Key Insight

> **A real bug violates a protocol law.**  
> **A spec bug violates an execution law OR a causal law.**

Once you classify the law being violated, the correct action becomes obvious.

---

## Final Sanity Gate

Before closing any CE diagnosis:

| Check | Status |
|-------|--------|
| CE survives full execution-closure check | ☐ |
| CE survives full causal-closure check | ☐ |
| No HAVOC in critical path | ☐ |
| No silent trust assumption | ☐ |
| No constraint hides exploit | ☐ |
| CE trace reproducible on-chain (in principle) | ☐ |
| Verdict is unambiguous | ☐ |
| If SPEC BUG: exact broken law identified | ☐ |
| If SPEC BUG: fix validated via removal test | ☐ |
| If REAL BUG: exploit path documented | ☐ |

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | [DATE] | Added bounded state checks, NONDET refinements, filtered clause checks, multi-contract analysis, CVL 2.0 repair tools, structured templates |
| 1.0 | [DATE] | Initial framework |

---

## Ghost Havocing Diagnosis Guide (NEW in v1.5)

> **Source:** RareSkills Certora Book — Ghost Variables and Persistent Ghosts

### What Is Ghost Havocing?

When a ghost variable is "havoced," the Prover assigns it an arbitrary (random) value. This happens:

| Event | Regular Ghost | Persistent Ghost |
|-------|---------------|------------------|
| Start of verification | Havoced | Havoced |
| Unresolved external call | **Havoced** | Retains value |
| Transaction reverts | Reset to pre-call | **Retains value** |

### Symptoms of Ghost Havocing

- Counterexample shows a ghost with an impossible value (e.g., negative balance sum)
- Invariant base case passes but inductive step fails with unrealistic initial values
- Rule passes with `require ghost == 0` but fails without it
- Values "jump" between function calls in the CE trace

### Diagnosis Steps

1. **Check ghost initialization:** Does the ghost have `init_state axiom`?
   ```cvl
   ghost mathint myGhost {
       init_state axiom myGhost == 0;  // Required for invariants
   }
   ```

2. **Check for unresolved external calls:** Are there external calls not modeled in the methods block? These cause ALL regular ghosts to be havoced after the call.
   - **Fix:** Add summary for the external call, or link the target contract

3. **Check if `persistent ghost` is needed:** If the ghost tracks state across reverts (e.g., low-level CALL return codes), it must be persistent:
   ```cvl
   persistent ghost bool g_callFailed;
   ```

4. **Check Sload hooks:** Without Sload hooks, ghosts can start from unrealistic values:
   ```cvl
   hook Sload uint256 val balanceOf[KEY address a] {
       require g_sum >= to_mathint(val);
   }
   ```

5. **Do NOT use `persistent` as a quick fix:** If a regular ghost is havoced due to unresolved external calls, the correct fix is to model the external call (link contract or add summary), NOT to make the ghost persistent.

### Persistent Ghost Use Cases

| Use Case | Regular | Persistent | Why |
|----------|---------|------------|-----|
| Sum tracking | ✓ | | Standard storage hooks maintain state |
| CALL opcode return tracking | | ✓ | Outer function may revert after CALL |
| Assembly transfer verification | | ✓ | Low-level calls need cross-revert tracking |
| Token counter | ✓ | | Standard accumulation |

### Persistent Ghost + CALL Hook Template

```cvl
persistent ghost bool g_lowLevelCallFail;

hook CALL(uint gas, address to, uint value,
          uint argsOffset, uint argsLength,
          uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallFail = true;
    } else {
        g_lowLevelCallFail = false;
    }
}
```

**See:** `cvl-language-deep-dive.md` Sections 8-9 for complete ghost and hook reference.
