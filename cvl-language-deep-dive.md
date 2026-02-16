# CVL Language Deep Dive — Complete Reference

> **Framework Version:** v1.9 (Red Team Hardening)
> **Source:** Extracted from the RareSkills Certora Book (60,000+ words, 35 chapters, official Certora collaboration)
> **Purpose:** Fill every CVL language gap — from foundational semantics to advanced patterns used in production OpenZeppelin/Solmate/Solady specifications.

---

## Table of Contents

1. [Type System: `mathint`, Casting, and Overflow](#1-type-system-mathint-casting-and-overflow)
2. [Core Statements: `require`, `assert`, `satisfy`](#2-core-statements-require-assert-satisfy)
3. [Logical Operators: Implication `=>` and Biconditional `<=>`](#3-logical-operators-implication--and-biconditional-)
4. [Vacuous Truth and Tautology — Silent Killers](#4-vacuous-truth-and-tautology--silent-killers)
5. [Method Tags: `@withrevert`, `@norevert`, and `lastReverted`](#5-method-tags-withrevert-norevert-and-lastreverted)
6. [Environment Variables: `env`, `msg.sender`, `msg.value`](#6-environment-variables-env-msgsender-msgvalue)
7. [Native Balances and `currentContract`](#7-native-balances-and-currentcontract)
8. [Ghost Variables — Complete Reference](#8-ghost-variables--complete-reference)
9. [Hooks — Sstore, Sload, and Opcode Hooks](#9-hooks--sstore-sload-and-opcode-hooks)
10. [`definition` Blocks — Reusable CVL Expressions](#10-definition-blocks--reusable-cvl-expressions)
11. [Invariants — Base Case, Inductive Step, and Preserved Blocks](#11-invariants--base-case-inductive-step-and-preserved-blocks)
12. [The `requireInvariant` Lifecycle](#12-the-requireinvariant-lifecycle)
13. [Parametric Rules and Method Properties](#13-parametric-rules-and-method-properties)
14. [Partially Parametric Rules and `helperSoundFnCall`](#14-partially-parametric-rules-and-helpersoundfncall)
15. [The Liveness / Effect / No-Side-Effect Pattern](#15-the-liveness--effect--no-side-effect-pattern)
16. [DISPATCHER and External Callback Resolution](#16-dispatcher-and-external-callback-resolution)
17. [Loops: Hidden Dangers and Configuration](#17-loops-hidden-dangers-and-configuration)
18. [Self-Transfer and Edge Case Handling](#18-self-transfer-and-edge-case-handling)
19. [Invariant Sanity Checks](#19-invariant-sanity-checks)
20. [Quick Reference Tables](#20-quick-reference-tables)

---

## 1. Type System: `mathint`, Casting, and Overflow

### Why `mathint` Exists

CVL arithmetic defaults to `mathint` — an **unbounded integer type** with no overflow or underflow. This is the single most important safety feature of CVL because it exposes arithmetic issues that Solidity's bounded types would silently wrap.

**Certora's rule of thumb:** *"Use `mathint` whenever possible; only use `uint` or `int` types for data that will be passed as input to contract functions."*

### `mathint` Catches What Solidity Misses

When a Solidity function uses an `unchecked` block or inline assembly, arithmetic wraps silently. CVL's `mathint` default catches this:

```cvl
rule average_overflow() {
    uint256 x;
    uint256 y;

    // returnVal comes from Solidity (bounded, may overflow)
    mathint returnVal = average(x, y);

    // Right-hand side evaluates as mathint (unbounded, correct)
    assert returnVal == (x + y) / 2;
    // FAILS when x + y > max_uint256 — overflow detected!
}
```

The right-hand side `(x + y) / 2` evaluates to `mathint` by default, correctly computing `2^255` when `x = 2^256 - 3, y = 3`. The left-hand side (Solidity's return) wraps to `0`. The mismatch exposes the bug.

### The Danger of `require_uint256`

`require_uint256` casts a `mathint` down to `uint256`, telling the Prover to **discard any counterexample** where the value exceeds the `uint256` range:

```cvl
// DANGEROUS — silently hides overflow
rule average_overflowIgnored() {
    uint256 x;
    uint256 y;
    mathint returnVal = average(x, y);

    uint256 numerator = require_uint256(x + y);     // ← overflow hidden
    uint256 expectedVal = require_uint256(numerator / 2);

    assert returnVal == expectedVal;  // PASSES — but the bug is hidden!
}
```

**Never use `require_uint256` on arithmetic expressions you're trying to verify.** It silently removes the exact counterexamples you need to find bugs.

### When Casting IS Appropriate

Use `require_uint256` only when you genuinely need to pass a `mathint` value to a Solidity function (which requires `uint256`), and you've already proven the value fits:

```cvl
// SAFE — value is provably bounded
requireInvariant totalSupplyBounded();
uint256 inputAmount = require_uint256(someComputedValue);
deposit(e, inputAmount);
```

Use `assert_uint256` when you want to convert a boolean expression to 0/1 for arithmetic:

```cvl
// Convert boolean to uint for balance delta
assert_uint256(from != to ? 1 : 0)
```

### Built-in Max Constants

CVL provides a complete family of built-in constants for maximum values of each integer type. These are available without any import or declaration:

| Constant | Value | Common Use |
|----------|-------|------------|
| `max_uint8` | $2^8 - 1 = 255$ | Enum bounds, small counters |
| `max_uint16` | $2^{16} - 1 = 65535$ | Fee basis points |
| `max_uint32` | $2^{32} - 1$ | Timestamps (pre-2106) |
| `max_uint40` | $2^{40} - 1$ | Extended timestamps |
| `max_uint64` | $2^{64} - 1$ | Rate accumulators |
| `max_uint128` | $2^{128} - 1$ | Half-slot packing |
| `max_uint256` | $2^{256} - 1$ | Standard balance/amount bounds |

> **Note:** Use the built-in constants directly (`max_uint128`) rather than defining your own (`definition MAX_UINT128 returns mathint = 2^128 - 1`). The built-ins are recognized by the Prover and guaranteed correct.

### Casting Quick Reference

| Cast | Behavior | Safety |
|------|----------|--------|
| `to_mathint(x)` | Widen `uint256` → `mathint` | Always safe |
| `require_uint256(x)` | Narrow `mathint` → `uint256`, **discards out-of-range counterexamples** | Dangerous — can hide bugs |
| `assert_uint256(x)` | Narrow `mathint` → `uint256`, **fails if out-of-range** | Safe — rejects impossible states |

---

## 2. Core Statements: `require`, `assert`, `satisfy`

### The Three Pillars

Every CVL rule must end with either `assert` or `satisfy`. These three statements control what the Prover explores:

| Statement | Meaning | Prover Behavior |
|-----------|---------|-----------------|
| `require P` | "Only consider states where P holds" | Filters out execution paths where P is false |
| `assert Q` | "Q must hold in ALL remaining paths" | Searches for counterexamples; rule fails if any found |
| `satisfy R` | "At least ONE path must make R true" | Searches for witnesses; rule fails if none found |

### `assert` — Universal Quantification ("For All")

Assert says: *"No matter what values the Prover picks, this condition must hold."*

```cvl
rule increment_increases_count() {
    uint256 before = count();
    increment();
    uint256 after = count();
    assert after == before + 1;  // Must hold for ALL starting states
}
```

If the Prover finds even one starting state where the assertion fails, it generates a **counterexample**.

### `satisfy` — Existential Quantification ("There Exists")

Satisfy says: *"Find me at least one set of values that makes this true."*

```cvl
rule canReachTen() {
    increment();
    increment();
    increment();
    satisfy count() == 10;  // Find ANY starting state where this holds
}
```

The Prover finds `count_initial = 7` → after 3 increments → `count = 10`. Rule is **verified** (witness found).

### `satisfy` as a Constraint Solver

The Prover can solve systems of equations using `satisfy`:

```solidity
// Solidity
function eqn(uint256 x, uint256 y) external pure returns (bool) {
    return (2 * x + 3 * y == 22) && (4 * x - y == 2);
}
```

```cvl
rule checkEqn(uint256 x, uint256 y) {
    satisfy eqn(x, y) == true;
    // Prover finds: x = 2, y = 6 ✓
}
```

If the system is inconsistent (parallel lines), the Prover correctly reports **violated** — no solution exists.

### Multiple `satisfy` Statements

If a rule has multiple `satisfy` statements, ALL must hold in at least one execution path. If any `satisfy` is on a conditional branch that isn't executed, it doesn't need to hold.

### Critical Rule: Rules MUST End With `assert` or `satisfy`

This will NOT compile:

```cvl
// COMPILE ERROR — no terminal assert or satisfy
rule broken() {
    uint256 x;
    if (x > 5) {
        assert x > 3;
    }
    // Missing terminal statement!
}
```

Fix: add `assert true;` as a fallback (hacky), or restructure using `=>` (idiomatic).

---

## 3. Logical Operators: Implication `=>` and Biconditional `<=>`

### Implication: `P => Q` ("If P, then Q")

The implication operator replaces the `if (P) { assert Q; }` pattern, which doesn't compile because the rule would lack a terminal assertion:

```cvl
// WON'T COMPILE — no terminal assert
rule broken() {
    if (x < y) { assert result == x; }
}

// CORRECT — use implication
rule correct() {
    assert x < y => result == x;
}
```

**Key property:** `P => Q` is TRUE whenever P is false (regardless of Q). This is not a bug — it means "when P doesn't apply, we make no claim."

### Implication is NOT Reversible

`P => Q` does **not** mean `Q => P`:

```cvl
// P => Q: "If x > y, then max returns x" ✓ VERIFIED
assert x > y => result == x;

// Q => P: "If max returns x, then x > y" ✗ FAILS
assert result == x => x > y;
// Counterexample: x = 3, y = 3 → result = x but x is NOT > y
```

Fix with `>=`:

```cvl
assert result == x => x >= y;  // ✓ VERIFIED
```

### Contrapositive: `P => Q` ≡ `!Q => !P`

These are logically equivalent — use whichever is more intuitive:

```cvl
// Original: "If x > y, max returns x"
assert x > y => result == x;

// Contrapositive: "If max didn't return x, then x ≤ y"
assert result != x => x <= y;  // Same truth, different perspective
```

### Biconditional: `P <=> Q` ("P if and only if Q")

The biconditional asserts BOTH directions simultaneously: `P => Q` AND `Q => P`.

```cvl
// Using two implications (verbose)
assert success => xAfter > xBefore;
assert xAfter > xBefore => success;

// Using biconditional (concise, stronger)
assert success <=> xAfter > xBefore;
```

**Why `<=>` is stronger than `=>`:** Implication says "if this condition holds, the outcome follows." Biconditional adds: "and NOTHING ELSE can cause this outcome."

### When to Use Each

| Use `=>` When... | Use `<=>` When... |
|---|---|
| You only care about one direction | You want to exhaustively prove conditions |
| Many possible causes for the outcome | There's a single condition that governs the outcome |
| You're isolating one revert scenario | You're enumerating ALL revert scenarios |
| Complex functions with many internal calls | Simple functions with clear preconditions |

### The Revert Biconditional Pattern

The most powerful use of `<=>` — exhaustively listing every condition that causes a revert:

```cvl
rule mulDivUp_allConditions_revert() {
    uint256 x; uint256 y; uint256 denominator;
    mulDivUp@withrevert(x, y, denominator);
    assert lastReverted <=> (denominator == 0 || x * y > max_uint256);
}
```

This proves: the function reverts **if and only if** one of these conditions holds. No other input can cause a revert.

---

## 4. Vacuous Truth and Tautology — Silent Killers

### Vacuous Truth: When P Is Always False

If the condition P in `P => Q` can never be true, the entire implication is trivially verified — regardless of Q:

```cvl
rule vacuously_true() {
    uint256 x;
    assert x < 0 => 1 == 2;  // VERIFIED! But meaningless.
}
```

Since `uint256` can never be negative, `x < 0` is unreachable. The Prover reports "verified" because there's no counterexample — not because `1 == 2` is true.

**This is the #1 source of false confidence in CVL specifications.**

### How Vacuous Truth Sneaks In

Common real-world scenarios where vacuous truth hides bugs:

```cvl
// Overly restrictive require makes assertion vacuous
rule dangerous() {
    uint256 x;
    require x > max_uint256;  // Impossible! No uint256 exceeds max_uint256
    assert somethingWrong();  // VERIFIED — vacuously
}
```

```cvl
// require filtering out ALL execution paths
rule also_dangerous(env e) {
    require e.msg.sender == address(0);  // address(0) can't send transactions
    require e.msg.sender != address(0);  // Contradicts above
    // All paths filtered — any assert passes vacuously
    assert false;  // VERIFIED!
}
```

### Tautology: When Q Is Always True

If the outcome Q is always true regardless of P, the implication is trivially verified:

```cvl
rule tautologically_true() {
    uint256 x; uint256 y;
    mathint result = max(x, y);
    assert x > y => result >= 0;  // VERIFIED — but result >= 0 is ALWAYS true for uint256
}
```

This doesn't prove anything about `max()` — it merely restates that unsigned integers are non-negative.

### Defenses Against Vacuity and Tautology

1. **Use `--rule_sanity basic`** (default since Prover v8.1.0) — the Prover adds automatic checks
2. **Use `satisfy`** to confirm reachability before writing `assert`:
   ```cvl
   rule check_reachability() {
       uint256 x;
       require x > 100;
       satisfy x > 100;  // Confirm this state IS reachable
   }
   ```
3. **Prefer `<=>` over `=>`** — biconditional forces you to account for all conditions
4. **Check the Prover report** for "vacuous" warnings in the sanity panel

---

## 5. Method Tags: `@withrevert`, `@norevert`, and `lastReverted`

### Default Behavior: Reverts Are Ignored

By default, the Prover **ignores all revert paths**. A function call without a tag is equivalent to `@norevert`:

```cvl
uint256 c = add(a, b);       // Reverts ignored
uint256 c = add@norevert(a, b);  // Same — reverts ignored
```

The Prover only considers paths where the function completes successfully.

### `@withrevert`: Include Revert Paths

Adding `@withrevert` tells the Prover to include execution paths that revert:

```cvl
uint256 c = add@withrevert(a, b);
// Now the Prover also tests inputs that would overflow
```

**When a revert occurs, return values are unconstrained** — `c` can be any arbitrary value after a revert.

### `lastReverted`: Track Whether a Revert Occurred

After a `@withrevert` call, `lastReverted` is `true` if the call reverted, `false` otherwise:

```cvl
rule addShouldRevert() {
    uint256 a; uint256 b;
    require a + b > max_uint256;  // Force overflow

    add@withrevert(a, b);
    assert lastReverted;  // Must revert on overflow
}

rule addShouldNotRevert() {
    uint256 a; uint256 b;
    require a + b <= max_uint256;  // No overflow

    add@withrevert(a, b);
    assert !lastReverted;  // Must NOT revert
}
```

### Critical: `lastReverted` Gets Overwritten

`lastReverted` updates after **every** function call. If you make a second call, the first revert status is lost:

```cvl
rule checkMath() {
    uint256 a;

    // CORRECT: capture immediately
    divide@withrevert(a, 0);
    bool divideReverted = lastReverted;  // ← capture NOW

    add@withrevert(a, 0);
    bool addReverted = lastReverted;

    assert divideReverted == true;   // Division by zero → revert
    assert addReverted == false;     // Adding zero → no revert
}
```

Without capturing `lastReverted` immediately, the second call overwrites it.

### The Biconditional Revert Pattern

The industry-standard pattern for exhaustive revert verification:

```cvl
rule function_revert(env e) {
    // Capture all pre-state needed
    uint256 balance = balanceOf(e.msg.sender);

    myFunction@withrevert(e, args);

    // Enumerate ALL revert conditions with <=>
    assert lastReverted <=> (
        e.msg.value != 0 ||           // non-payable
        balance < amount ||            // insufficient balance
        e.msg.sender == address(0)     // zero-address caller
    );
}
```

This proves: the function reverts **if and only if** one of the listed conditions holds.

### The `bool success` Pattern

For readability, capture revert status as a named boolean:

```cvl
rule transferOwnership(env e) {
    address newOwner;
    address current = owner();

    transferOwnership@withrevert(e, newOwner);
    bool success = !lastReverted;

    assert success <=> (e.msg.sender == current && newOwner != 0);
    assert success => owner() == newOwner;
}
```

---

## 6. Environment Variables: `env`, `msg.sender`, `msg.value`

### When Functions Need `env`

Functions that depend on Solidity globals (`msg.sender`, `msg.value`, `block.timestamp`, etc.) require an `env` parameter in CVL. Functions that don't depend on any globals are marked `envfree`.

```cvl
methods {
    function owner() external returns (address) envfree;     // No globals needed
    function counter() external returns (uint256) envfree;   // No globals needed
    // increment() NOT declared — it depends on msg.sender, so it's env-dependent
}
```

**Rule:** If a function is environment-dependent, it does NOT need to be in the methods block (unless adding `envfree`, `optional`, or a summary).

### Declaring `env` in Rules

Two equivalent styles:

```cvl
// Style 1: env as rule parameter
rule myRule(env e) {
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
}

// Style 2: env declared in rule body
rule myRule() {
    env e;
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
}
```

### Passing `env` to Functions

Environment-dependent functions MUST receive `e` as the first argument:

```cvl
increment@withrevert(e);          // ✓ Correct
increment@withrevert(e, args);    // ✓ With additional args
increment@withrevert();           // ✗ COMPILE ERROR — env required
```

### The Non-Payable Trap

Non-payable functions revert when `msg.value != 0`. The Prover WILL find this:

```cvl
// This rule FAILS — the Prover sends ETH to a non-payable function
rule naiveAccessControl(env e) {
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
    // Counterexample: msg.sender IS owner, but msg.value = 1 → revert!
}

// Fixed — exclude non-payable revert
rule fixedAccessControl(env e) {
    require e.msg.value == 0;  // Non-payable function
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
}
```

### Standard Non-Payable Precondition

Use a `definition` for cleanliness:

```cvl
definition nonpayable(env e) returns bool = e.msg.value == 0;

rule myRule(env e) {
    require nonpayable(e);
    // ...
}
```

### The Overflow Trap

The Prover also finds overflows that cause reverts:

```cvl
// FAILS — counter at max_uint256 causes overflow revert
rule stillFails(env e) {
    require e.msg.value == 0;
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
    // Counterexample: sender IS owner, but counter == max_uint256 → revert!
}

// Fixed — exclude counter overflow
rule fullyFixed(env e) {
    require e.msg.value == 0;
    require counter() < max_uint256;
    increment@withrevert(e);
    assert !lastReverted <=> e.msg.sender == owner();
}
```

### Relaxing With `=>` Instead of `<=>`

If you only want to prove "non-owners always revert" without accounting for ALL revert conditions, use implication:

```cvl
rule simpler(env e) {
    increment@withrevert(e);
    assert e.msg.sender != owner() => lastReverted;
    // No preconditions needed — just proving one direction
}
```

---

## 7. Native Balances and `currentContract`

### `nativeBalances[address]`

CVL provides a built-in mapping for querying the native ETH balance of any address:

```cvl
mathint ethBalance = nativeBalances[someAddress];
mathint contractBalance = nativeBalances[currentContract];
```

### `currentContract`

Refers to the contract being verified:

```cvl
// Check that the contract's ETH balance increased
mathint balanceBefore = nativeBalances[currentContract];
deposit(e);
mathint balanceAfter = nativeBalances[currentContract];
assert balanceAfter == balanceBefore + e.msg.value;
```

### The Self-Call Problem

When `msg.sender == currentContract`, ETH sent "to itself" doesn't change the net balance, but WETH still gets minted. This breaks naive assertions:

```cvl
// FAILS when msg.sender == currentContract
rule deposit_naive(env e) {
    mathint before = nativeBalances[currentContract];
    deposit(e);
    mathint after = nativeBalances[currentContract];
    assert after == before + e.msg.value;
    // Counterexample: contract deposits to itself → no net ETH change but WETH minted
}

// Fixed
rule deposit_correct(env e) {
    require e.msg.sender != currentContract;  // Exclude self-call
    mathint before = nativeBalances[currentContract];
    deposit(e);
    mathint after = nativeBalances[currentContract];
    assert after == before + e.msg.value;
}
```

### ETH Balance as a Revert Condition

The Prover discovers insufficient ETH balance as a hidden revert condition:

```cvl
// FAILS — Prover finds case where sender doesn't have enough ETH
rule register_naive(env e) {
    register@withrevert(e);
    assert !lastReverted <=> e.msg.value >= 5 * 10^16;
    // Counterexample: msg.value = 1 ETH but nativeBalances[sender] = 0
}

// Fixed
rule register_fixed(env e) {
    require nativeBalances[e.msg.sender] >= e.msg.value;
    register@withrevert(e);
    assert !lastReverted <=> e.msg.value >= 5 * 10^16;
}
```

**CVL exponentiation uses `^`**, not Solidity's `**`.

---

## 8. Ghost Variables — Complete Reference

### What Ghosts Are

Ghost variables are specification-level storage that persists across a rule's execution. They bridge the gap between CVL rules (which can only see public functions) and Solidity's private storage.

### Declaration

```cvl
// Simple ghost
ghost mathint totalVotes;

// Ghost mapping
ghost mapping(address => mathint) balanceTracker;

// Ghost with initial state axiom (for invariants)
ghost mathint sumOfBalances {
    init_state axiom sumOfBalances == 0;
}

// Ghost with global axiom (always-true constraint)
ghost mathint positiveCounter {
    axiom positiveCounter >= 0;
}
```

### Havocing — The Core Challenge

At the start of verification, ghost variables hold **arbitrary (havoced) values**. The Prover intentionally assigns random values to test all possible states:

```cvl
ghost mathint callCount;

rule broken() {
    increment(); increment(); increment();
    assert callCount == 3;
    // FAILS — callCount started at -2, ended at 1
}

rule fixed() {
    require callCount == 0;  // Anchor the ghost
    increment(); increment(); increment();
    assert callCount == 3;  // ✓ VERIFIED
}
```

### When Ghosts Get Havoced

| Event | Regular Ghost | Persistent Ghost |
|-------|---------------|------------------|
| Start of verification | Havoced | Havoced |
| Unresolved external call | Havoced | **Retains value** |
| Transaction reverts | Reset to pre-call state | **Retains value** |
| Between rules | Havoced | Havoced |

### Persistent Ghosts

Declared with `persistent ghost` — they survive unresolved external calls and reverts:

```cvl
persistent ghost bool g_lowLevelCallFailed;

hook CALL(uint gas, address to, uint value,
          uint argsOffset, uint argsLength,
          uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallFailed = true;
    } else {
        g_lowLevelCallFailed = false;
    }
}
```

**Warning:** Do NOT use `persistent` as a quick fix when verification fails due to havoc from unresolved external calls. The correct fix is to link the actual contract implementation into the verification scene.

### Initialization: `init_state axiom` vs `axiom`

| Axiom Type | When It Applies | Use Case |
|------------|----------------|----------|
| `init_state axiom sumOfBal == 0` | Only before the constructor | Initialize ghosts to match deployment state |
| `axiom counter >= 0` | Every state throughout verification | Universal constraints that always hold |

### Constraining Ghosts

**In rules:** Use `require` to set initial ghost values:
```cvl
rule myRule() {
    require ghostSum == 0;
    // ...
}
```

**In invariants:** Use `init_state axiom` — `require` CANNOT initialize ghosts for invariants because invariants must hold unconditionally:
```cvl
ghost mathint totalVotes {
    init_state axiom totalVotes == 0;
}
```

---

## 9. Hooks — Sstore, Sload, and Opcode Hooks

### The Problem Hooks Solve

CVL rules can only interact through a contract's public interface. Many properties depend on private storage behavior — hooks intercept storage operations to bridge this gap.

### Sstore Hook — Triggered on Storage Writes

**Basic syntax:**
```cvl
hook Sstore count uint256 newValue (uint256 oldValue) {
    ghostCount = newValue;
}
```

**Mapping syntax (with key capture):**
```cvl
hook Sstore balanceOf[KEY address account] uint256 newVal (uint256 oldVal) {
    sumOfBalances = sumOfBalances + newVal - oldVal;
}
```

**Old value capture is optional:**
```cvl
hook Sstore balanceOf[KEY address account] uint256 newVal {
    // Only new value available
}
```

### The Delta Pattern — Sum Tracking

The standard pattern for tracking aggregate sums across a mapping:

```cvl
ghost mathint g_sumOfBalances {
    init_state axiom g_sumOfBalances == 0;
}

hook Sstore balanceOf[KEY address account] uint256 newVal (uint256 oldVal) {
    g_sumOfBalances = g_sumOfBalances + newVal - oldVal;
}
```

`+ newVal - oldVal` correctly handles all cases:
- Balance increases: `+100 - 50 = +50`
- Balance decreases: `+50 - 100 = -50`
- No change: `+100 - 100 = 0`

### Ownership Tracking via Sstore Hook

For ERC-721, tracking mint/burn/transfer in a single hook:

```cvl
ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + (newOwner != 0 ? 1 : 0);
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - (oldOwner != 0 ? 1 : 0);
}
```

The ternary `(addr != 0 ? 1 : 0)` handles:
- **Mint:** oldOwner = 0 (no decrement), newOwner ≠ 0 (increment)
- **Burn:** oldOwner ≠ 0 (decrement), newOwner = 0 (no increment)
- **Transfer:** both ≠ 0 (decrement old, increment new)

### Sload Hook — Triggered on Storage Reads

Sload hooks constrain values when they're read from storage, preventing the Prover from using unrealistic initial values:

```cvl
hook Sload uint256 balance balanceOf[KEY address addr] {
    require g_sumOfBalances >= to_mathint(balance);
}
```

**Why Sload is preferred over Sstore for constraints:**

- **Sstore blind spot:** If the Prover never writes to a slot, the Sstore constraint is never checked. The Prover can still start from a weird initial value.
- **Sload advantage:** Runs every time a balance is **read**. Even if a slot was never written to, reading it triggers the constraint. Acts as a **global sanity check**.

### The Unchecked Block Problem

In `unchecked` blocks, the Prover explores overflow paths by assigning unrealistic initial values:

```solidity
// Solidity — unchecked addition
unchecked { pointsOf[_user] += _amount; }
```

Without a Sload hook, the Prover can:
1. Set `pointsOf[user] = 0x2` initially
2. Add `2^256 - 2` to wrap around to `0`
3. Create an impossible state where `pointsOf[user] > totalPoints`

The Sload hook prevents this by constraining every read.

### CALL Opcode Hook — For Assembly Verification

Captures the return code of low-level `CALL` operations:

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

Must be `persistent` because the outer function may revert after the CALL, and we need to retain the call's return status.

### Hook Scope Limitation

Variables declared inside hooks are **local to the hook block**. They cannot be accessed from rules. Use ghost variables to bridge the gap.

---

## 10. `definition` Blocks — Reusable CVL Expressions

### Syntax

```cvl
definition nonpayable(env e) returns bool = e.msg.value == 0;
definition balanceLimited(address account) returns bool = balanceOf(account) < max_uint256;
definition authSanity(env e) returns bool = e.msg.sender != 0;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
```

### Usage

```cvl
rule transferOwnership(env e) {
    require nonpayable(e);         // Reusable non-payable check
    require balanceLimited(to);    // Prevent unrealistic overflow
    // ...
}
```

### `definition` vs CVL Functions

| Feature | `definition` | CVL function |
|---------|-------------|--------------|
| Syntax | `definition name(...) returns type = expr;` | `function name(...) returns type { ... }` |
| Body | Single expression | Multi-line with local variables |
| Use case | Simple boolean predicates | Complex logic with branching |

### Standard Definitions Library

These appear repeatedly in OpenZeppelin, Solmate, and Solady specifications:

```cvl
// Non-payable constraint
definition nonpayable(env e) returns bool = e.msg.value == 0;

// Prevent unrealistic balances (for unchecked blocks)
definition balanceLimited(address account) returns bool =
    balanceOf(account) < max_uint256;

// Exclude zero-address as transaction sender
definition nonzerosender(env e) returns bool = e.msg.sender != 0;

// Sender authentication sanity
definition authSanity(env e) returns bool = e.msg.sender != 0;
```

---

## 11. Invariants — Base Case, Inductive Step, and Preserved Blocks

### How Invariant Verification Works

The Prover uses **mathematical induction** to verify invariants:

1. **Base Case ("After the constructor"):** The invariant must hold immediately after the constructor finishes
2. **Inductive Step ("After external methods"):** For every public/external function — if the invariant holds before the call, it must still hold after

```cvl
invariant totalVotesMatch()
    to_mathint(totalVotes()) == votesInFavor() + votesAgainst();
```

The Prover expands this to:
- **Step 1:** Deploy contract → check `0 == 0 + 0` ✓
- **Step 2:** Assume invariant holds → call `voteInFavor()` → check invariant still holds ✓
- **Step 3:** Assume invariant holds → call `voteAgainst()` → check invariant still holds ✓

View/pure functions are automatically skipped (they can't change state).

### Preserved Blocks

When the Prover explores the inductive step, it starts from any symbolic state where the invariant holds. This can include unrealistic states that cause false violations. Preserved blocks add assumptions to constrain the inductive step:

**Generic preserved block (applies to all functions):**
```cvl
invariant solvency()
    nativeBalances[currentContract] >= totalSupply()
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
        require balanceOf(e.msg.sender) <= totalSupply();
    }
}
```

**Function-specific preserved block:**
```cvl
invariant tokenIntegrity()
    nativeBalances[currentContract] >= totalSupply()
{
    preserved deposit() with (env e) {
        require e.msg.sender != currentContract;
    }
    preserved withdraw(uint256 amount) with (env e) {
        require e.msg.sender != currentContract;
        require balanceOf(e.msg.sender) <= totalSupply();
    }
}
```

**Constructor preserved block (Prover v8.5.1+):**
```cvl
invariant timestampPositive()
    deployedAt() > 0
{
    preserved constructor() with (env e) {
        require e.block.timestamp != 0;
    }
}
```

### Preserved Block Best Practices

- **Prefer preserved over filtered**: Filters remove functions from checking entirely (riskier). Preserved blocks just constrain assumptions.
- **Self-call elimination**: `require e.msg.sender != currentContract` is almost always needed.
- **Balance bounding**: `require balanceOf(e.msg.sender) <= totalSupply()` prevents unrealistic symbolic states where individual balance > total supply.
- **Generic blocks can consolidate**: If assumptions are global truths, merge multiple function-specific blocks into one generic block.

### Filter Blocks

Restrict which methods are checked in the inductive step:

```cvl
invariant totalSupplyIntegrity()
    to_mathint(totalSupply()) == sumOfBalances
    filtered {
        f -> !f.isView
    }
```

---

## 12. The `requireInvariant` Lifecycle

### The Problem

Every new rule starts from a fresh symbolic state with arbitrary values. Even if you've proven `sumOfBalances == totalSupply` as an invariant, the Prover does NOT automatically assume it holds when checking other rules:

```cvl
// This rule starts with arbitrary balances — invariant not assumed!
rule transfer_effect(env e, address to, uint256 amount) {
    // balanceOf(sender) could be > totalSupply in the Prover's symbolic state
    // This causes false violations in unchecked arithmetic
}
```

### The Lifecycle

1. **Phase 1: Use `require` during development**
   ```cvl
   rule transfer_effect(env e, address to, uint256 amount) {
       require balanceOf(e.msg.sender) + balanceOf(to) <= max_uint256;
       // ... rule logic
   }
   ```
   This gets the rule passing, but `require` is an unverified assumption.

2. **Phase 2: Prove the invariant**
   ```cvl
   invariant totalSupplyEqualsSumOfBalances()
       to_mathint(totalSupply()) == g_sumOfBalances;
   ```

3. **Phase 3: Upgrade `require` → `requireInvariant`**
   ```cvl
   rule transfer_effect(env e, address to, uint256 amount) {
       requireInvariant totalSupplyEqualsSumOfBalances();
       // Now the precondition is PROVEN, not assumed
   }
   ```

### `require` vs `requireInvariant`

| Feature | `require` | `requireInvariant` |
|---------|-----------|-------------------|
| Source of truth | Writer's assertion | Previously proven property |
| Can hide bugs? | Yes — wrong conditions are silently accepted | No — references a verified invariant |
| Documentation | Manual | Self-documenting — links to the invariant |

### Using `requireInvariant` in Invariants

Invariants can depend on other invariants via preserved blocks:

```cvl
invariant sumProperty()
    to_mathint(totalSupply()) == sumOfBalances;

invariant individualCap(address a)
    to_mathint(balanceOf(a)) <= to_mathint(totalSupply())
{
    preserved with (env e) {
        requireInvariant sumProperty();  // Depend on proven property
    }
}
```

### Invariant Chains

Complex specifications build **layers of invariants**, each proven independently and then reused:

```
Layer 1: totalSupplyEqualsSumOfBalances
    ↓ (used in preserved block of)
Layer 2: individualBalanceCap
    ↓ (used in preserved block of)
Layer 3: solvencyInvariant
```

---

## 13. Parametric Rules and Method Properties

### Parametric Rule Anatomy

A parametric rule tests a property across ALL public/external functions:

```cvl
rule totalSupplyIsConstant() {
    env e;
    mathint beforeSupply = totalSupply(e);

    method f;
    calldataarg args;
    f(e, args);  // Execute ANY function

    mathint afterSupply = totalSupply(e);
    assert beforeSupply == afterSupply;
}
```

The Prover creates one rule instance per function and tests each independently.

### Scope: One Call Only

Parametric rules analyze the effect of **one function call** in isolation. They don't test sequences of calls.

### Method Properties

Accessible on the `method` variable:

| Property | Returns | Use Case |
|----------|---------|----------|
| `f.selector` | 4-byte selector | Identify specific functions |
| `f.isView` | `bool` | Skip view functions |
| `f.isPure` | `bool` | Skip pure functions |
| `f.isFallback` | `bool` | Identify fallback/receive |
| `f.numberOfArguments` | `uint` | Filter by argument count |
| `f.contract` | Contract reference | Multi-contract verification |

### Comparing Selectors

```cvl
if (f.selector == sig:mint(address,uint256).selector) {
    assert supplyAfter >= supplyBefore;
}
else if (f.selector == sig:burn(uint256).selector) {
    assert supplyAfter <= supplyBefore;
}
else {
    assert supplyAfter == supplyBefore;
}
```

### Filtered Blocks in Parametric Rules

```cvl
rule totalSupplyIntegrityCheck(env e, method f, calldataarg args) filtered {
    f -> !f.isView  // Skip view functions
} {
    mathint supplyBefore = totalSupply(e);
    f(e, args);
    mathint supplyAfter = totalSupply(e);
    // assertions...
}
```

### Authorization Parametric Rules

"Only certain functions can change state X":

```cvl
rule onlyMethodsCanChangeTotalSupply(env e, method f, calldataarg args) {
    mathint before = totalSupply(e);
    f(e, args);
    mathint after = totalSupply(e);

    assert after != before => (
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:burn(uint256).selector
    );
}
```

---

## 14. Partially Parametric Rules and `helperSoundFnCall`

### The Problem

In a fully parametric rule, `f(e, args)` executes every function with completely arbitrary arguments. But some functions need specific preconditions to avoid false violations (e.g., `mint` needs `balanceLimited(to)`). Applying these preconditions to ALL functions would over-constrain paths that don't need them.

### The Solution: `helperSoundFnCall`

A helper function that routes each function to its specific preconditions:

```cvl
function helperSoundFnCall(env e, method f) {
    if (f.selector == sig:mint(address,uint256).selector) {
        address to; uint256 tokenId;
        require balanceLimited(to);
        mint(e, to, tokenId);
    }
    else if (f.selector == sig:burn(uint256).selector) {
        uint256 tokenId;
        requireInvariant ownerHasBalance(tokenId);
        burn(e, tokenId);
    }
    else if (f.selector == sig:transferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        transferFrom(e, from, to, tokenId);
    }
    else {
        calldataarg args;
        f(e, args);  // Default: arbitrary args, no constraints
    }
}
```

### Usage in Rules

```cvl
rule stateChangeRestriction(env e) {
    require nonzerosender(e);

    mathint supplyBefore = _supply;

    method f;
    helperSoundFnCall(e, f);  // Sound routing instead of f(e, args)

    mathint supplyAfter = _supply;

    assert supplyAfter > supplyBefore => (
        supplyAfter == supplyBefore + 1 &&
        f.selector == sig:mint(address,uint256).selector
    );
}
```

### Why "Sound"?

Preconditions are applied **only where needed**. The `else` fallback passes completely arbitrary arguments — ensuring maximum coverage for functions without special requirements.

---

## 15. The Liveness / Effect / No-Side-Effect Pattern

### The Industry Standard

OpenZeppelin uses this three-part assertion pattern for every function verification:

#### 1. Liveness — "When does it succeed?"

```cvl
assert success <=> (ownerBefore == 0 && to != 0);
```

A **biconditional** proving the function succeeds if and only if the listed conditions hold.

#### 2. Effect — "What changes on success?"

```cvl
assert success => (
    _supply == supplyBefore + 1 &&
    to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
    unsafeOwnerOf(tokenId) == to
);
```

An **implication** proving that on success, all expected state changes occur.

#### 3. No Side Effect — "Did anything else change?"

```cvl
assert balanceOf(otherAccount) != balanceOfOtherBefore => otherAccount == to;
assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId;
```

**Implications** proving that any state change implies it's the targeted entity.

### Complete Rule Template

```cvl
rule mint(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address to; uint256 tokenId;

    // Preconditions
    require balanceLimited(to);
    requireInvariant ownerHasBalance(tokenId);

    // Pre-state snapshot
    address ownerBefore = unsafeOwnerOf(tokenId);
    mathint supplyBefore = _supply;
    mathint balanceOfToBefore = balanceOf(to);

    // Other accounts for side-effect checking
    address otherAccount;
    uint256 otherTokenId;
    mathint balanceOfOtherBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    mint@withrevert(e, to, tokenId);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (ownerBefore == 0 && to != 0);

    // EFFECT
    assert success => (
        _supply == supplyBefore + 1 &&
        to_mathint(balanceOf(to)) == balanceOfToBefore + 1 &&
        unsafeOwnerOf(tokenId) == to
    );

    // NO SIDE EFFECT
    assert balanceOf(otherAccount) != balanceOfOtherBefore => otherAccount == to;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId;
}
```

---

## 16. DISPATCHER and External Callback Resolution

### The Problem

When a contract calls an external function (like `IERC721Receiver.onERC721Received`), the Prover can't resolve where the call goes. This triggers **AUTO-havoc** — all computed values are overwritten with arbitrary values, making post-call assertions meaningless.

### The Solution: DISPATCHER

```cvl
methods {
    function _.onERC721Received(address, address, uint256, bytes) external
        => DISPATCHER(true);
}
```

This tells the Prover: "Route this call to all contracts in the verification scene that implement this method."

### Mock Receiver Contract (Dispatchee)

A minimal contract that the DISPATCHER can route to:

```solidity
// ERC721ReceiverHarness.sol
import {IERC721Receiver} from "../IERC721Receiver.sol";

contract ERC721ReceiverHarness is IERC721Receiver {
    function onERC721Received(
        address, address, uint256, bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC721Received.selector;  // Always accept
    }
}
```

Include this contract in the verification scene (certoraRun command).

### Liveness Catches Callback Failures

Interesting insight: you don't need to explicitly model callback failure scenarios. The Liveness assertion (`success <=> conditions`) catches them implicitly — when the receiver callback reverts, `success` is false but the right-hand-side conditions remain true, violating the biconditional.

---

## 17. Loops: Hidden Dangers and Configuration

### The Prover vs Loops

Loops with symbolic bounds create **path explosion** — each conditional inside a loop doubles the execution paths ($2^n$ for $n$ iterations).

### Configuration Flags

| Flag | Behavior | Soundness |
|------|----------|-----------|
| `--loop_iter N` | Unroll loops N times (default: 1) | **Incomplete** — only proves for inputs up to N |
| `--optimistic_loop` | Assume property holds beyond the bound | **Unsound** — makes unverified assumptions |

Neither provides a sound AND complete proof for unbounded loops.

### Hidden Loops — The String Trap

Solidity strings create **4 hidden loops** at the bytecode level:
1. `calldata` → `memory` copy (function entry)
2. `memory` → `storage` copy (assignment like `txt = _txt`)
3. `storage` → `memory` copy (reading via getter)
4. Word-by-word comparison (equality check)

If you see "unwinding condition in a loop" errors on code without visible loops, check for strings, dynamic arrays, or bytes.

### Configuration

```json
{
    "files": ["contracts/MyContract.sol:MyContract"],
    "verify": "MyContract:specs/myspec.spec",
    "optimistic_loop": true,
    "loop_iter": "3"
}
```

---

## 18. Self-Transfer and Edge Case Handling

### The Self-Transfer Edge Case

When `from == to` in a transfer, the balance should remain unchanged:

```cvl
rule transfer_effect(env e) {
    address to; uint256 amount;
    address sender = e.msg.sender;

    mathint senderBefore = balanceOf(sender);
    mathint receiverBefore = balanceOf(to);

    transfer(e, to, amount);

    if (sender == to) {
        // Self-transfer: balance unchanged
        assert to_mathint(balanceOf(sender)) == senderBefore;
    } else {
        assert to_mathint(balanceOf(sender)) == senderBefore - amount;
        assert to_mathint(balanceOf(to)) == receiverBefore + amount;
    }
}
```

### The Compact Pattern

Using `assert_uint256` for elegant handling:

```cvl
assert success => (
    to_mathint(balanceOf(from)) == balFromBefore - assert_uint256(from != to ? 1 : 0) &&
    to_mathint(balanceOf(to))   == balToBefore   + assert_uint256(from != to ? 1 : 0)
);
```

### Harness Functions for Edge Cases

When public functions revert on edge cases (e.g., `ownerOf()` reverts for unminted tokens), create harness functions that return zero instead:

```solidity
// Harness — returns address(0) instead of reverting
function unsafeOwnerOf(uint256 tokenId) external view returns (address) {
    return _owners[tokenId];  // Direct storage access, no revert
}

function unsafeGetApproved(uint256 tokenId) external view returns (address) {
    return _tokenApprovals[tokenId];
}
```

This is essential because invariants must evaluate to boolean values, not revert.

---

## 19. Invariant Sanity Checks

### Automatic Sanity Checks (Prover v8.1.0+)

The Prover automatically runs sanity checks with `--rule_sanity basic` (default):

| Check | What It Verifies |
|-------|------------------|
| `rule_not_vacuous` | The invariant is evaluated under realistic conditions — preconditions don't eliminate all paths |
| `invariant_not_trivial_postcondition` | The invariant's condition isn't trivially true (e.g., `true` or `uint256 >= 0`) |

### Interpreting Results

These appear in the Prover UI as expandable entries under each invariant. They're not rules you wrote — they're automatic quality checks.

- **Green ✓ on sanity checks:** Your invariant is meaningful
- **Red ✗ on `rule_not_vacuous`:** Your preconditions filtered out all execution paths — the invariant is vacuously true
- **Red ✗ on `invariant_not_trivial_postcondition`:** Your invariant condition is always true regardless of state

### Configuration

```json
{
    "rule_sanity": "basic"     // default since v8.1.0
    // "rule_sanity": "advanced"  // more thorough
    // "rule_sanity": "none"      // disable sanity checks
}
```

---

## 19.1 Built-in Rules — Automated Safety Checks (NEW — Prover v8.8.0)

Built-in rules are general-purpose verification rules provided by the Certora Prover. They check common vulnerability patterns **without writing any contract-specific rules**. Enable them with `use builtin rule <name>;` in your spec file.

### Complete Built-in Rule Reference

| Rule | Syntax | What It Checks | Conf Extras |
|------|--------|----------------|-------------|
| `sanity` | `use builtin rule sanity;` | At least one non-reverting path per function | None |
| `deepSanity` | `use builtin rule deepSanity;` | Reachability of interesting program points (if/else branches, external calls) | `--multi_assert_check` |
| `msgValueInLoopRule` | `use builtin rule msgValueInLoopRule;` | Detects `msg.value` usage or delegate calls inside loops | None |
| `hasDelegateCalls` | `use builtin rule hasDelegateCalls;` | Detects any delegate call in any function | None |
| `viewReentrancy` | `use builtin rule viewReentrancy;` | Read-only reentrancy: ensures view functions return consistent values during external calls | None |
| `safeCasting` | `use builtin rule safeCasting;` | Whether any Solidity cast can be out of bounds (e.g., `uint16(x)` where `x > 2^16 - 1`) | `"safe_casting_builtin": true` |
| `uncheckedOverflow` | `use builtin rule uncheckedOverflow;` | Whether `+`, `-`, `*` in `unchecked` blocks can overflow/underflow | `"unchecked_overflow_builtin": true` |

### `uncheckedOverflow` — Catching Unsafe Unchecked Arithmetic

Solidity `unchecked` blocks disable overflow/underflow protection for gas optimization. The `uncheckedOverflow` builtin rule automatically verifies that every arithmetic operation (`+`, `-`, `*`) inside these blocks is actually safe.

```cvl
// In your .spec file:
use builtin rule uncheckedOverflow;
```

```json
// In your .conf file — REQUIRED:
{
    "unchecked_overflow_builtin": true
}
```

**How it works:** For each external method, the Prover analyzes every reachable unchecked arithmetic operation. If any combination of inputs can cause overflow/underflow, the rule fails for that method+operation pair.

**Example — this will fail:**
```solidity
function basicMul(uint128 x, uint128 y) public pure returns (uint) {
    unchecked {
        return x * y;  // uint128 * uint128 CAN overflow uint128
    }
}
```

> **When to use:** Run `uncheckedOverflow` early in every verification project. It catches exactly the bugs that `mathint` catches in CVL — but at the Solidity level, automatically, without writing a single rule.

### `safeCasting` — Catching Unsafe Type Narrowing

Solidity's explicit cast operators (like `uint16(x)`) silently truncate values without reverting. The `safeCasting` builtin rule automatically verifies that every reachable cast operation is in bounds.

```cvl
// In your .spec file:
use builtin rule safeCasting;
```

```json
// In your .conf file — REQUIRED:
{
    "safe_casting_builtin": true
}
```

**What it catches:**
- Unsigned narrowing: `uint16(x)` where `x > 2^16 - 1`
- Signed/unsigned: `int16(x)` where `x > 2^15 - 1` (for unsigned to signed)
- Signed narrowing: `int8(x)` where `x < -128` or `x > 127`

> **Relationship to `require_uint256`:** This is the Solidity-level complement to CVL's `require_uint256` danger. While `require_uint256` masks overflow in specs, `safeCasting` catches truncation in the contract itself.

### `--assume_no_casting_overflow` — Suppressing Cast Warnings

```json
// In your .conf file:
{
    "assume_no_casting_overflow": true
}
```

This flag tells the Prover to **assume** all casts are in bounds. Use it only when:
- You've already verified casts are safe (via `safeCasting` or manual review)
- Cast range failures are causing noise in unrelated rules
- You want to focus verification on non-casting properties

> ⚠️ **This is an underapproximation** — Solidity does NOT revert on out-of-bounds casts. Using this flag hides real truncation bugs. Always run `safeCasting` first before suppressing.

### Recommended Builtin Rule Strategy

```cvl
// Phase 0 — Run BEFORE writing any custom rules:
use builtin rule sanity;            // Can all methods execute?
use builtin rule uncheckedOverflow;  // Any unsafe unchecked math?
use builtin rule safeCasting;        // Any unsafe casts?

// Phase 1 — Run alongside custom rules:
use builtin rule viewReentrancy;     // Read-only reentrancy?
use builtin rule msgValueInLoopRule;  // msg.value in loops?
use builtin rule hasDelegateCalls;    // Unexpected delegates?
```

```json
// Corresponding .conf extras:
{
    "unchecked_overflow_builtin": true,
    "safe_casting_builtin": true
}
```

---

## 20. Quick Reference Tables

### Statement Semantics

| Statement | Quantifier | Prover searches for | Rule passes if |
|-----------|------------|-----------|---------------|
| `assert Q` | Universal (∀) | Counterexample where Q is false | No counterexample found |
| `satisfy R` | Existential (∃) | Witness where R is true | At least one witness found |
| `require P` | Filter | N/A — constrains search space | N/A |

### Operator Reference

| Operator | Syntax | Meaning | When to use |
|----------|--------|---------|-------------|
| Implication | `P => Q` | If P then Q | One-directional assertions |
| Biconditional | `P <=> Q` | P if and only if Q | Exhaustive condition lists |
| Contrapositive | `!Q => !P` | Equivalent to `P => Q` | Alternative perspective |

### Ghost Initialization Summary

| Context | Method | Example |
|---------|--------|---------|
| In rules | `require` | `require ghostSum == 0;` |
| In invariants | `init_state axiom` | `ghost mathint x { init_state axiom x == 0; }` |
| Universal constraint | `axiom` | `ghost mathint x { axiom x >= 0; }` |

### Hook Types

| Hook | Trigger | Captures | Primary Use |
|------|---------|----------|-------------|
| `Sstore var` | Storage write | `newValue (oldValue)` | Track state changes |
| `Sload var` | Storage read | `value` | Constrain initial values |
| `CALL(...)` | EVM CALL opcode | `returnCode` | Track low-level call results |

### Standard Definitions Library

```cvl
definition nonpayable(env e) returns bool = e.msg.value == 0;
definition balanceLimited(address a) returns bool = balanceOf(a) < max_uint256;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
```

### Four-Phase Verification Methodology

| Phase | Question | Pattern |
|-------|----------|---------|
| **1. Correctness** | "Does the function do what it should?" | Liveness (`<=>`) + Effect (`=>`) |
| **2. Side Effects** | "Does it change anything it shouldn't?" | No-Side-Effect assertions on uninvolved accounts |
| **3. Invariants** | "Does any global property break?" | `invariant` with ghost + hook infrastructure |
| **4. Authorization** | "Can only permitted actors trigger changes?" | Parametric rule with `f.selector` checks |

---

*This document is part of the Certora-Fv-Framework v1.9 (Red Team Hardening). Knowledge sourced from the RareSkills Certora Book — a collaboration between RareSkills and Certora.*
