# Certora Specification Framework — Execution-Closed & Causally-Closed Universe

> **Version:** 3.1 (Adversarial Verification Loop + Validation Evidence Gate)  
> **CVL Syntax Reference:** CVL 2.0 (Certora Prover)  
> **Philosophy:** You are not verifying a contract. You are verifying a *closed EVM universe* with that contract inside it.

**v3.0 Companion Documents:**
- **impact-spec-template.md** — Economic impact tracking & anti-invariants ⭐ NEW v3.0
- **multi-step-attacks-template.md** — Flash loan, sandwich, staged attack patterns ⭐ NEW v3.0
- **cvl-language-deep-dive.md** — Complete CVL language reference (types, ghosts, hooks, invariants) ⭐ NEW v1.5
- **verification-playbooks.md** — Production-ready worked examples (ERC-20, WETH, ERC-721) ⭐ NEW v1.5
- **best-practices-from-certora.md** — Official tutorial techniques for invariants, CE investigation, loop handling
- **advanced-cli-reference.md** — Performance optimization, timeout mitigation ⭐ NEW v1.4
- **quick-reference-v1.3.md** — Printable cheat sheet with CVL syntax and phase checklists
- **categorizing-properties.md Section 7** — Property prioritization (HIGH/MEDIUM/LOW)

---

## Table of Contents

1. [Non-Negotiable Axioms](#non-negotiable-axioms)
2. [Non-Negotiable Principles](#non-negotiable-principles)
3. [Modeling Strategy Rules](#modeling-strategy-rules)
4. [Spec Header Template](#spec-header-template)
5. [Phase -2: Scope Definition](#phase--2-scope-definition)
6. [Phase -1: Execution Surface Closure](#phase--1-execution-surface-closure)
7. [Phase 0: Causal Closure Validation](#phase-0-causal-closure-validation)
8. [Phase 1: Foundation Layer (Ghosts, Hooks, Valid State)](#phase-1-foundation-layer)
9. [Phase 2: Property Classification](#phase-2-property-classification)
10. [Phase 3: Property Implementation](#phase-3-property-implementation)
11. [Phase 4: Final Validation Checklist](#phase-4-final-validation-checklist)
12. [CVL 2.0 Templates](#cvl-20-templates)

---

## Non-Negotiable Axioms

### EXECUTION CLOSURE AXIOM

> Every external call, external storage read, or external view function that can influence control-flow, state, or assertions MUST be explicitly modeled (dispatcher, constant summary, or justified assumption).
>
> Unmodeled external interactions are treated as adversarial HAVOC and **invalidate the spec**.

### CREATION CLOSURE REQUIREMENT

> Any contract creation (CREATE / CREATE2 / factory deployment / clone) that can influence the verified contract's storage MUST be:
>
> * explicitly modeled (including constructor effects), or
> * explicitly forbidden via `assume` or `require`.

**Constructor Causality Clause:**

> If storage written during a constructor is later read (directly or indirectly) by the verified contract, those constructor writes are part of the execution surface and MUST be modeled or constrained.
>
> Unmodeled contract creation or constructor writes are equivalent to HAVOC on all constructor-initialized storage and invalidate any dependent invariant or rule.

### CAUSAL CLOSURE AXIOM

> For every state variable (contract storage or ghost) referenced in any invariant or rule:
>
> * All execution paths that can mutate that variable MUST be explicitly enumerated and modeled.
> * Every such mutation MUST be causally linked to a modeled call, constructor, or hook.
>
> If a variable can change without passing through a modeled mutation path, the specification is invalid.
>
> **Surface closure without causal closure is UNSOUND.**

### BOUNDED STATE AXIOM

> Every unbounded collection (array, mapping key space, timestamp, block number) MUST have an explicit bound that is:
>
> * Realistic for on-chain usage
> * Documented with justification
> * Applied in a `validState()` function or rule precondition
>
> Common bounds (from authoritative Certora specs):
> * Array lengths: `< 1000` (or smaller based on gas limits)
> * Token IDs / Request IDs: `< 100000000`
> * Timestamps: `<= max_uint40` (year ~36812)
> * Loop iterations for invariant preservation: `<= 5`

### ADDRESS CLOSURE RULE

> Any `address` that represents a contract, token, oracle, or registry MUST be:
>
> * immutable, **or**
> * bound to a modeled contract, **or**
> * constrained via `assume` with justification.
>
> **Unconstrained contract-typed addresses are HAVOC.**

> Any external address returned by a function (`underlying()`, `factory()`, etc.) MUST be:
>
> * linked to a concrete contract, or
> * constrained immediately via `assume`
>
> Any address assumed to be a contract MUST also have:
>
> * bound executable code (dispatcher / summary), or
> * an explicitly forbidden call surface.

---

## Non-Negotiable Principles

* On-chain code is ground truth (including full execution surface)
* Documentation expresses **intent only**, never guarantees
* Separate intent, assumptions, enforced properties, modeled execution reality
* Refuse to write properties before execution closure
* Refuse to write properties before causal closure validation
* Mark every invariant with its **source of truth**

---

## Modeling Strategy Rules

### NONDET Summary Rules

**NONDET is ACCEPTABLE for:**

| Scenario | Justification |
|----------|---------------|
| View functions whose return value doesn't influence safety-critical control flow | Return value is informational only |
| State-changing functions that only affect their own contract's state (not the verified contract) | Isolated side effects |
| Functions on contracts we don't control and whose behavior is irrelevant to the property | Out of scope for this verification |

**NONDET is FORBIDDEN for:**

| Scenario | Why |
|----------|-----|
| Functions that modify the verified contract's storage | Unmodeled state mutation = HAVOC |
| Functions whose return value determines branch outcomes in safety-critical paths | Control flow depends on unmodeled value |
| Functions that can trigger callbacks into the verified contract | Reentrancy path unmodeled |

**Example from authoritative specs:**
```cvl
// ✅ ACCEPTABLE: View function, return doesn't affect safety
function _.getResumeSinceTimestamp() external => NONDET;

// ✅ ACCEPTABLE: Only affects external contract's state
function _.transferOwnership(address newOwner) external => NONDET;

// ❌ FORBIDDEN: Modifies verified contract via callback
function _.onTokenTransfer(address, uint256, bytes) external => NONDET; // WRONG!
```

### DISPATCHER Rules

Use `DISPATCHER(true)` when:
* Multiple implementations exist (e.g., different Escrow instances)
* You want the prover to route calls to known implementations
* The function is called on addresses that could be any of several contracts

```cvl
// Route to known implementations
function _.getRageQuitSupport() external => DISPATCHER(true);
function _.startRageQuit(Duration, Duration) external => DISPATCHER(true);
```

### Custom Summary Rules

Use custom summaries when:
* You need deterministic behavior for uninterpreted functions
* You need to prevent havoc from complex external calls

> 🚨 **Custom Summary Accuracy Validation (NEW v1.9):**
> Custom summaries are **trusted by construction** — the Prover will not verify that
> your summary matches the real function's behavior. For each custom summary, you MUST:
>
> 1. **Document** whether the summary is an exact model, overapproximation, or underapproximation
> 2. **Justify** any determinism assumption (uninterpreted ghost = same input → same output).
>    If the real function is non-deterministic (e.g., reads mutable storage), an uninterpreted
>    ghost overpromises and may hide real bugs.
> 3. **Add a checklist entry** in `spec_authoring.md` Phase -1.3 Modeling Obligations:
>    `| [function] | Custom Summary | [Exact/Over/Under] | [Justification] |`
> 4. **If the summary constrains return values** (e.g., always returns `true`, caps a price),
>    verify that removing the constraint does not reintroduce a real exploit.

```cvl
// Ghost-based uninterpreted function (same input = same output)
// ACCURACY: Overapproximation — assumes deterministic behavior.
// JUSTIFICATION: Real function reads immutable storage; determinism is code-enforced.
ghost CVLCallGetResumeSinceTimestampBool(address) returns bool; 
ghost CVLCallGetResumeSinceTimestampStamp(address) returns uint256; 

function CVLCallGetResumeSinceTimestamp(address sealable) returns (bool, uint256) {
    return (CVLCallGetResumeSinceTimestampBool(sealable),
            CVLCallGetResumeSinceTimestampStamp(sealable));
}

// In methods block:
function _.callGetResumeSinceTimestamp(address sealable) internal 
    => CVLCallGetResumeSinceTimestamp(sealable) expect (bool, uint256);
```

### Read/Write Pairing Rule

> Every external read MUST have a modeled write, OR proof of immutability.

| Read Function | Must Model Write(s) |
|---------------|---------------------|
| `balanceOf(address)` | `transfer`, `mint`, `burn`, `transferFrom` |
| `totalSupply()` | `mint`, `burn` |
| `owner()` | `transferOwnership`, constructor |
| `allowance(address,address)` | `approve`, `increaseAllowance`, `decreaseAllowance` |
| `getPrice()` | oracle update mechanism OR `assume` with justification |

🚫 If a read is not paired with modeled writes → **STOP and model first**.

---

## Spec Header Template

```cvl
/**
 * ================================================================
 * CERTORA SPECIFICATION — EXECUTION-CLOSED & CAUSALLY-CLOSED UNIVERSE
 * ================================================================
 *
 * Contract Under Verification: [CONTRACT_NAME]
 * CVL Version: 2.0
 *
 * Governing Axioms:
 *   - Execution Closure Axiom
 *   - Causal Closure Axiom  
 *   - Bounded State Axiom
 *   - Address Closure Rule
 *   - On-chain code is ground truth
 *
 * Source Documents (NON-ENFORCING):
 *   - [CONTRACT_DOCUMENTATION]
 *
 * Execution Surface:
 *   - Primary: [CONTRACT_NAME]
 *   - External: [LIST_OF_EXTERNAL_CONTRACTS]
 *
 * IMPORTANT:
 *   This header is documentary.
 *   Execution is defined ONLY by explicit CVL constructs.
 *   Causal closure is proven, not assumed.
 *
 * Counterexample Interpretation:
 *   - HAVOC-based counterexamples → modeling defects
 *   - Unmodeled mutation path counterexamples → causal closure defects
 *   - Unbounded value counterexamples → missing bounds
 * ================================================================
 */
```

---

## Phase -2: Scope Definition

> **This phase MUST be completed before any other work.**

### Step 1: Enumerate All Contracts in Call Graph

Create a table listing every contract that interacts with the verified contract:

| Contract | Relationship | Modeling Strategy |
|----------|--------------|-------------------|
| `[PrimaryContract]` | Under verification | Full modeling |
| `[ERC20Token]` | Called for transfers | DISPATCHER or harness |
| `[Oracle]` | Price reads | `assume` with justification |
| `[Registry]` | Address lookups | CONSTANT or DISPATCHER |
| `[Callback]` | Can call back | Must model reentrancy |

### Step 2: Enumerate All External Functions

For **each external contract**, create this table:

| Contract | Function | Type | Modeling Decision | Justification |
|----------|----------|------|-------------------|---------------|
| ERC20 | `transfer(address,uint256)` | State-changing | DISPATCHER | Multiple token implementations |
| ERC20 | `balanceOf(address)` | View | DISPATCHER | Tracks real balances |
| Oracle | `getPrice()` | View | `assume` | Trust assumption documented |
| Registry | `getAddress(bytes32)` | View | CONSTANT | Immutable after deployment |

### Step 3: Document Address Bindings

| Address Variable | Source | Binding |
|-----------------|--------|---------|
| `token` | Constructor parameter | Bound to `DummyERC20` |
| `oracle` | `getOracle()` return | Constrained via `assume` |
| `factory` | Immutable | Bound to `Factory` contract |

---

## Phase -1: Execution Surface Closure

> **This phase MUST be completed before causal validation.**

### Methods Block Structure

```cvl
methods {
    // ═══════════════════════════════════════════════════════════════
    // SECTION 1: ENVFREE DECLARATIONS
    // Functions that don't depend on msg.sender, msg.value, or block.*
    // ═══════════════════════════════════════════════════════════════
    function ContractName.pureFunction() external returns (uint256) envfree;
    function ContractName.viewFunction() external returns (address) envfree;
    
    // ═══════════════════════════════════════════════════════════════
    // SECTION 2: DISPATCHER SUMMARIES
    // For functions with multiple implementations
    // ═══════════════════════════════════════════════════════════════
    function _.transferableFunction() external => DISPATCHER(true);
    
    // ═══════════════════════════════════════════════════════════════
    // SECTION 3: NONDET SUMMARIES (with justification)
    // ═══════════════════════════════════════════════════════════════
    // JUSTIFICATION: View function, return doesn't affect verified contract
    function _.externalViewFunction() external => NONDET;
    
    // JUSTIFICATION: Only affects external contract's own state
    function _.externalStateChange() external => NONDET;
    
    // ═══════════════════════════════════════════════════════════════
    // SECTION 4: CUSTOM SUMMARIES
    // For complex behaviors requiring deterministic modeling
    // ═══════════════════════════════════════════════════════════════
    function ComplexContract.complexFunction(address, bytes) 
        external returns (bytes) => CVLComplexFunction();
    
    // ═══════════════════════════════════════════════════════════════
    // SECTION 5: CONSTANT SUMMARIES
    // For truly immutable values
    // ═══════════════════════════════════════════════════════════════
    function _.immutableValue() external => CONSTANT;
}
```

### Custom Summary Implementations

```cvl
// For functions returning bytes (to avoid havoc)
function nondetBytes() returns bytes {
    bytes b;
    return b;
}

// For uninterpreted functions (deterministic: same input = same output)
ghost myUninterpretedFunction(address, uint256) returns uint256;

function CVLMyFunction(address a, uint256 x) returns uint256 {
    return myUninterpretedFunction(a, x);
}
```

---

## Phase 0: Causal Closure Validation

> **This phase MUST be completed BEFORE writing any property CVL.**  
> **This phase PROVES causal closure, doesn't assume it.**

### Property Analysis Template

For EACH property, document this analysis:

```markdown
## Property: [PROPERTY_NAME]
**Type:** [INVARIANT/RULE]
**Source:** [Security property document reference]

### Variables Involved:
| Variable | Type | Location | Purpose in Property |
|----------|------|----------|---------------------|
| `balances[user]` | `uint256` | Storage mapping | User's token balance |
| `totalSupply` | `uint256` | Storage | Sum of all balances |
| `sumBalances` | `mathint` | Ghost | Tracks sum for verification |

### Mutation Paths Analysis:
**`balances[user]`:**
| Function | Effect | Modeled? | Hook? |
|----------|--------|----------|-------|
| `transfer(to, amount)` | Decreases sender, increases receiver | ✅ | ✅ Sstore hook |
| `mint(to, amount)` | Increases receiver | ✅ | ✅ Sstore hook |
| `burn(from, amount)` | Decreases holder | ✅ | ✅ Sstore hook |
| `constructor()` | Initial zero | ✅ | init_state axiom |

**`totalSupply`:**
| Function | Effect | Modeled? | Hook? |
|----------|--------|----------|-------|
| `mint(to, amount)` | Increases | ✅ | ✅ Sstore hook |
| `burn(from, amount)` | Decreases | ✅ | ✅ Sstore hook |
| `constructor()` | Initial zero | ✅ | init_state axiom |

### Causal Closure Verified: ✅
All mutation paths enumerated and modeled.
```

### CVL 2.0 Validation Rules

**Mutation Path Completeness (Parametric Rule):**

```cvl
/**
 * @title Mutation Path Analysis for [PROPERTY_NAME]
 * @notice Verifies all changes to [variable] go through expected functions
 * @dev If this fails, there's an unmodeled mutation path
 */
rule mutation_paths_[variable_name](method f) 
    filtered { 
        f -> f.selector != sig:irrelevantFunction().selector 
    } 
{
    // Capture state before
    uint256 valueBefore = relevantVariable;
    
    env e;
    calldataarg args;
    f(e, args);
    
    uint256 valueAfter = relevantVariable;
    
    // If value changed, it must be through an expected mutator
    assert valueBefore != valueAfter => (
        f.selector == sig:expectedMutator1(address,uint256).selector ||
        f.selector == sig:expectedMutator2(address).selector ||
        f.selector == sig:expectedMutator3().selector
    ), "Unmodeled mutation path detected for [variable]";
}
```

**Ghost Synchronization Verification:**

```cvl
/**
 * @title Ghost Synchronization for sumBalances
 * @notice Verifies ghost stays synchronized with storage
 */
rule ghost_synchronization_sumBalances(method f, address anyUser) 
    filtered { f -> f.contract == currentContract }
{
    // Require the invariant holds before
    requireInvariant totalIsSumBalances();
    
    env e;
    calldataarg args;
    f(e, args);
    
    // Ghost must still equal the actual sum
    // (This is implicitly checked by the invariant, 
    //  but we make it explicit here for validation)
    assert to_mathint(totalSupply()) == sumBalances,
        "Ghost desynchronized from storage";
}
```

**Constructor Initialization Verification:**

```cvl
/**
 * @title Constructor Initialization for [PROPERTY_NAME]
 * @notice Verifies initial state satisfies the property
 */
rule constructor_establishes_[property_name] {
    // Start from a fresh state (constructor just ran)
    require totalSupply() == 0;
    require sumBalances == 0;
    
    // Verify the property holds initially
    assert to_mathint(totalSupply()) == sumBalances,
        "Property not established by constructor";
}
```

**Invariant Preservation with Dependencies:**

```cvl
/**
 * @title Preservation test for [INVARIANT_NAME]
 * @notice Explicitly tests that dependent invariants are needed
 */
rule preservation_test_[invariant_name](method f, address user) 
    filtered { f -> f.contract == currentContract }
{
    // Load all dependent invariants
    requireInvariant dependentInvariant1();
    requireInvariant dependentInvariant2(user);
    
    // Assume the invariant holds before
    require [invariant_condition_before];
    
    env e;
    calldataarg args;
    f(e, args);
    
    // Assert it holds after
    assert [invariant_condition_after],
        "Invariant [NAME] not preserved by function";
}
```

---

## Phase 1: Foundation Layer

> **Only proceed if Phase 0 validation passes.**

### Ghost Variables

```cvl
// ═══════════════════════════════════════════════════════════════
// GHOST DECLARATIONS
// Only for aggregates — never mirror individual owned storage
// ═══════════════════════════════════════════════════════════════

/**
 * @title Sum of all balances
 * @notice Tracks aggregate that has no on-chain equivalent
 * @dev Updated by Sstore hooks on balances mapping
 */
ghost mathint sumBalances {
    // Constructor initializes to zero
    init_state axiom sumBalances == 0;
}

/**
 * @title Count of non-zero balance holders  
 * @notice For invariants about holder count
 */
ghost mathint holderCount {
    init_state axiom holderCount == 0;
}

/**
 * @title Mirror for array length (needed for loop bounds)
 */
ghost uint256 arrayLengthMirror {
    init_state axiom arrayLengthMirror == 0;
}
```

### Impact Category Ghosts (NEW v3.0 — Offensive Verification)

> **Purpose:** First-class primitives for economic impact tracking and attack discovery.
> **Reference:** See `impact-spec-template.md` for complete infrastructure.

```cvl
// ═══════════════════════════════════════════════════════════════
// IMPACT CATEGORY GHOSTS — Required for offensive verification
// ═══════════════════════════════════════════════════════════════

/**
 * @title Per-Actor Value Tracking
 * @notice Tracks total value controlled by each address
 * @dev Essential for detecting attacker profit
 */
persistent ghost mapping(address => mathint) actor_value {
    init_state axiom forall address a. actor_value[a] == 0;
}

/**
 * @title Total System Value
 * @notice Sum of all value held by protocol contracts
 * @dev For detecting value leakage
 */
ghost mathint total_system_value {
    init_state axiom total_system_value == 0;
}

/**
 * @title Value Extraction Counter
 * @notice Cumulative value extracted from protocol
 * @dev Only increases; represents potential attacker profit
 */
ghost mathint total_value_extracted {
    init_state axiom total_value_extracted == 0;
}

/**
 * @title Insolvency Flag
 * @notice True if protocol obligations exceed holdings
 */
ghost bool insolvent_state {
    init_state axiom insolvent_state == false;
}

/**
 * @title Share Dilution Factor
 * @notice Tracks unexpected inflation of claims/shares
 * @dev Non-zero indicates dilution attack possible
 */
ghost mathint dilution_factor {
    init_state axiom dilution_factor == 0;
}

/**
 * @title Debt Socialization Tracking
 * @notice Tracks losses pushed onto innocent users
 */
ghost mapping(address => mathint) socialized_loss {
    init_state axiom forall address a. socialized_loss[a] == 0;
}

/**
 * @title Liquidity Freeze Flag
 * @notice True if users blocked from withdrawing
 */
ghost bool liquidity_frozen {
    init_state axiom liquidity_frozen == false;
}

/**
 * @title Irreversible Loss Flag
 * @notice True if extraction cannot be recovered
 */
ghost bool irreversible_loss_occurred {
    init_state axiom irreversible_loss_occurred == false;
}
```

### Impact Tracking Hooks (NEW v3.0)

```cvl
// ═══════════════════════════════════════════════════════════════
// IMPACT TRACKING HOOKS
// Adapt to your protocol's storage layout
// ═══════════════════════════════════════════════════════════════

/**
 * @title Balance Change Hook for Actor Value
 * @notice Updates actor_value when balances change
 * @dev Change `balances` to match your storage variable
 */
hook Sstore balances[KEY address account] uint256 newBal (uint256 oldBal) {
    mathint delta = to_mathint(newBal) - to_mathint(oldBal);
    actor_value[account] = actor_value[account] + delta;
}

/**
 * @title ETH Transfer Hook
 * @notice Tracks native ETH flows between addresses
 */
hook CALL(uint g, address target, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint retval {
    if (value > 0) {
        actor_value[currentContract] = actor_value[currentContract] - to_mathint(value);
        actor_value[target] = actor_value[target] + to_mathint(value);
    }
}
```

### Impact Definitions (NEW v3.0)

```cvl
// ═══════════════════════════════════════════════════════════════
// IMPACT DEFINITIONS
// Helper definitions for attack detection rules
// ═══════════════════════════════════════════════════════════════

/**
 * @title Profitable Attack Detection
 * @notice Returns true if actor gained value
 */
definition profitable_attack(address attacker, mathint before, mathint after) 
    returns bool = after > before;

/**
 * @title Harmful Extraction Detection
 * @notice Returns true if system lost value
 */
definition harmful_extraction(mathint sys_before, mathint sys_after) 
    returns bool = sys_after < sys_before;

/**
 * @title Position Delta
 * @notice Calculates change in actor's position
 */
definition position_delta(address actor, mathint before, mathint after) 
    returns mathint = after - before;
```

### Anti-Invariant Templates (NEW v3.0)

> **Philosophy:** These rules are EXPECTED TO FAIL. A failure = exploit found.
> If they pass, no attack path was discovered (or model is incomplete).

```cvl
// ═══════════════════════════════════════════════════════════════
// ANTI-INVARIANTS — Rules expected to FAIL
// Counterexample = exploit parameters
// ═══════════════════════════════════════════════════════════════

/**
 * @title Attacker Profit Search
 * @notice Searches for profitable function calls
 * @dev EXPECTED TO FAIL if exploit exists
 */
rule attacker_cannot_profit(env e, method f) 
    filtered { f -> !f.isView && !f.isFallback }
{
    require e.msg.sender != 0;
    require e.msg.value == 0;  // Adjust for payable
    
    address attacker = e.msg.sender;
    mathint value_before = actor_value[attacker];
    
    calldataarg args;
    f(e, args);
    
    mathint value_after = actor_value[attacker];
    
    // SHOULD FAIL if there's a profit path
    assert !profitable_attack(attacker, value_before, value_after), 
        "EXPLOIT: Caller profited from this call";
}

/**
 * @title System Value Conservation
 * @notice Verifies protocol value doesn't leak
 * @dev EXPECTED TO FAIL if value extractable
 */
rule system_value_conserved(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require e.msg.sender != 0;
    
    mathint system_before = total_system_value;
    
    calldataarg args;
    f@withrevert(e, args);
    
    mathint system_after = total_system_value;
    
    assert system_after == system_before, 
        "EXPLOIT: System value changed unexpectedly";
}

/**
 * @title Profit Path Discovery
 * @notice Uses satisfy to find profitable inputs
 * @dev If VERIFIED, witness contains attack parameters
 */
rule find_profitable_inputs(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    address attacker = e.msg.sender;
    mathint value_before = actor_value[attacker];
    
    calldataarg args;
    f@withrevert(e, args);
    
    mathint value_after = actor_value[attacker];
    mathint profit = value_after - value_before;
    
    // Find inputs that create profit
    satisfy profit > 0;
}
```

### Impact-Driven Invariants (NEW v3.0)

```cvl
// ═══════════════════════════════════════════════════════════════
// IMPACT INVARIANTS — Standard safety properties
// ═══════════════════════════════════════════════════════════════

/**
 * @title No Value Extraction
 * @notice Protocol should never have value extracted
 */
invariant no_value_extraction()
    total_value_extracted == 0

/**
 * @title Always Solvent
 * @notice Protocol must always be solvent
 */
invariant always_solvent()
    !insolvent_state

/**
 * @title No Share Dilution
 * @notice Share supply should not be inflatable
 */
invariant no_share_dilution()
    dilution_factor == 0

/**
 * @title No Liquidity Freeze
 * @notice Users must always be able to exit
 */
invariant no_liquidity_freeze()
    !liquidity_frozen
```

### Hooks

```cvl
// ═══════════════════════════════════════════════════════════════
// STORAGE HOOKS
// Every mutation path MUST have a corresponding hook
// ═══════════════════════════════════════════════════════════════

/**
 * @title Balance update hook
 * @notice Maintains sumBalances ghost synchronized with storage
 */
hook Sstore balances[KEY address user] uint256 newBalance (uint256 oldBalance) {
    sumBalances = sumBalances + newBalance - oldBalance;
    
    // Update holder count
    if (oldBalance == 0 && newBalance > 0) {
        holderCount = holderCount + 1;
    } else if (oldBalance > 0 && newBalance == 0) {
        holderCount = holderCount - 1;
    }
}

/**
 * @title Balance read hook  
 * @notice Enforces relationship between individual and aggregate
 */
hook Sload uint256 value balances[KEY address user] {
    // Individual balance cannot exceed the sum of all balances
    require value <= sumBalances;
}

/**
 * @title Array length hooks (for bounded iteration)
 */
hook Sload uint256 length currentContract.myArray.length {
    require length == arrayLengthMirror;
}

hook Sstore currentContract.myArray.length uint256 newLength {
    arrayLengthMirror = newLength;
}
```

### Valid State Function

```cvl
// ═══════════════════════════════════════════════════════════════
// VALID STATE BUNDLE
// Collects all invariants and bounds — call at start of every rule
// ═══════════════════════════════════════════════════════════════

/**
 * @title Valid State Function
 * @notice Bundles all protocol invariants and realistic bounds
 * @dev MUST be called at the start of every behavioral rule
 */
function validState() {
    // ─────────────────────────────────────────────────────────────
    // REALISTIC BOUNDS (from Bounded State Axiom)
    // ─────────────────────────────────────────────────────────────
    
    // Array bounds (gas limits make larger arrays impractical)
    require currentContract.myArray.length < 1000;
    
    // ID bounds (realistic for on-chain usage)
    require lastId < 100000000;
    require lastId > 0;  // Zero often means "not set"
    
    // ─────────────────────────────────────────────────────────────
    // INVARIANT CHAIN (order matters for dependencies)
    // ─────────────────────────────────────────────────────────────
    
    // Level 1: No dependencies
    requireInvariant zeroAddressHasNoBalance();
    requireInvariant totalSupplyIsNonNegative();
    
    // Level 2: Depends on Level 1
    requireInvariant sumBalancesEqualsTotalSupply();
    
    // Level 3: Depends on Level 2
    requireInvariant noBalanceExceedsTotalSupply();
}

/**
 * @title Environment Constraints
 * @notice Standard constraints for env parameter
 * @dev Call after validState() in rules that use env
 */
function validEnv(env e) {
    // Timestamp bounds
    require e.block.timestamp <= max_uint40;
    require e.block.timestamp > 0;
    
    // Contract cannot call itself (unless explicitly modeling reentrancy)
    require e.msg.sender != currentContract;
    
    // Realistic gas (optional, for gas-sensitive properties)
    // require e.msg.gas <= 30000000;
}
```

### Individual Invariants

```cvl
// ═══════════════════════════════════════════════════════════════
// INVARIANT DECLARATIONS
// Each invariant: one truth, with preserved blocks and filtering
// ═══════════════════════════════════════════════════════════════

/**
 * @title Zero address has no balance
 * @notice address(0) cannot hold tokens
 * @dev Level: 1 | Dependencies: NONE | Foundation invariant
 */
invariant zeroAddressHasNoBalance()
    balanceOf(0) == 0
    filtered { 
        // Exclude external contracts that can't affect this
        f -> f.contract == currentContract 
    }
    {
        preserved with (env e) {
            // Standard environment constraints
            require e.msg.sender != currentContract;
            require e.block.timestamp <= max_uint40;
        }
    }

/**
 * @title Sum of balances equals total supply
 * @notice Ghost sumBalances tracks the actual sum
 * @dev Level: 2 | Dependencies: zeroAddressHasNoBalance
 */
invariant sumBalancesEqualsTotalSupply()
    to_mathint(totalSupply()) == sumBalances
    filtered { 
        f -> f.contract == currentContract 
    }
    {
        preserved with (env e) {
            require e.msg.sender != currentContract;
            requireInvariant zeroAddressHasNoBalance();
        }
    }

/**
 * @title No single balance exceeds total supply
 * @notice Derived from sum invariant
 * @dev Level: 3 | Dependencies: sumBalancesEqualsTotalSupply
 */
invariant noBalanceExceedsTotalSupply(address user)
    balanceOf(user) <= totalSupply()
    filtered { 
        f -> f.contract == currentContract 
    }
    {
        preserved with (env e) {
            require e.msg.sender != currentContract;
            requireInvariant sumBalancesEqualsTotalSupply();
        }
    }

/**
 * @title Complex invariant with multiple dependencies
 * @notice Example of chaining requireInvariant
 */
invariant complexInvariant(uint256 id, address user)
    someCondition(id, user)
    filtered { 
        f -> f.contract == currentContract &&
             f.selector != sig:adminOverride().selector  // Trusted role
    }
    {
        preserved functionThatNeedsSpecialHandling(address a) with (env e) {
            // Load dependent invariants with correct parameters
            requireInvariant simpleInvariant();
            requireInvariant parameterizedInvariant(a);
            
            // Bound loop iterations for this specific function
            require currentContract.array.length <= 5;
        }
        
        preserved with (env e) {
            require e.msg.sender != currentContract;
            requireInvariant simpleInvariant();
        }
    }
```

---

## Phase 2: Property Classification

> **Before ANY property implementation, classify it.**

**See also:**
- **best-practices-from-certora.md Section 3** - Invariant design patterns (monotonicity, conservation laws)
- **categorizing-properties.md Section 7** - Property prioritization framework

### Classification Decision Tree

```
┌─────────────────────────────────────────────────────────────────┐
│ QUESTION 1: Is all state referenced by this property            │
│             owned by verified contract or modeled?              │
├─────────────────────────────────────────────────────────────────┤
│ ❌ No  → STOP. Complete Phase -1 and Phase 0 first.            │
│ ✅ Yes → Continue to Question 2                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ QUESTION 2: Is the question "Can this EVER happen?"             │
│             (regardless of how we got there)                    │
├─────────────────────────────────────────────────────────────────┤
│ ✅ Yes → INVARIANT                                              │
│ ❌ No  → Continue to Question 3                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ QUESTION 3: Does it depend on a specific function call,         │
│             caller identity, or call sequence?                  │
├─────────────────────────────────────────────────────────────────┤
│ ✅ Yes → RULE                                                   │
│ ❌ No  → Continue to Question 4                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ QUESTION 4: Would a single violation leave the protocol         │
│             irrecoverably broken?                               │
├─────────────────────────────────────────────────────────────────┤
│ ✅ Yes → INVARIANT (critical safety property)                   │
│ ❌ No  → Continue to Question 5                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ QUESTION 5: Does violation require a trusted role to            │
│             misbehave (admin, owner, governance)?               │
├─────────────────────────────────────────────────────────────────┤
│ ✅ Yes → OUT OF SCOPE (or model with filtered exclusion)        │
│ ❌ No  → RULE (behavioral property)                             │
└─────────────────────────────────────────────────────────────────┘
```

### Classification Record Template

```markdown
## Property Classification Record

**Property Name:** [NAME]
**Property Statement:** [Natural language description]

### Decision Path:
1. All state owned/modeled? ✅ Yes
2. "Can this EVER happen?" ❌ No (it's about what happens WHEN transfer is called)
3. Depends on specific function? ✅ Yes (transfer function)
4. N/A (already classified)
5. N/A (already classified)

**Classification:** RULE
**Justification:** Property describes behavior of a specific function (transfer), 
not a global state condition.
```

---

## Phase 3: Property Implementation

> **Only proceed if:**
> 1. Phase 0 causal validation passed
> 2. Property is classified
> 3. All dependencies are identified

### Invariant Implementation Pattern

```cvl
/**
 * @title [INVARIANT_NAME]
 * @notice [What this invariant guarantees]
 * @dev Dependencies: [list dependent invariants]
 *      Source: [security property document reference]
 */
invariant invariant_[name]([parameters])
    [condition]
    filtered { 
        // Exclude functions that cannot affect this property
        f -> f.contract == currentContract &&
             f.selector != sig:trustedAdminFunction().selector
    }
    {
        // Specific function handling (if needed)
        preserved specificFunction(address param) with (env e) {
            // Load dependencies
            requireInvariant dependency1();
            requireInvariant dependency2(param);
            
            // Environment constraints
            require e.msg.sender != currentContract;
            
            // Loop bounds (if function has loops)
            require currentContract.array.length <= 5;
        }
        
        // Default preservation block
        preserved with (env e) {
            // Always load dependencies
            requireInvariant dependency1();
            
            // Standard constraints
            require e.msg.sender != currentContract;
            require e.block.timestamp <= max_uint40;
        }
    }
```

### Rule Implementation Pattern

> **CRITICAL: By default, the Prover ignores revert paths.** If you call `f(e, args)` without
> `@withrevert`, any execution that reverts is silently pruned — your rule only proves the
> "happy path" and leaves failure conditions unverified.
>
> **Use `@withrevert` whenever you need to reason about WHY a function succeeds or fails.**
> Use the Liveness/Effect/No-Side-Effect pattern (Section 15 of cvl-language-deep-dive.md)
> as the standard for complete verification.

#### Pattern A: Success-Path-Only Rule (when failure behavior is verified separately)

```cvl
/**
 * @title [RULE_NAME] — Success Path
 * @notice [What this rule verifies about successful execution]
 * @dev Source: [security property document reference]
 *      Depends on invariants: [list]
 *      Revert conditions: Verified separately in rule_[name]_revert
 */
rule rule_[name](env e, [parameters]) 
    filtered {
        f -> f.contract == currentContract
    }
{
    // STEP 1: Load valid state (MANDATORY)
    validState();
    validEnv(e);
    
    // STEP 2: Preconditions (ONLY contract-enforced ones)
    require amount > 0;  // Only if Solidity has: require(amount > 0)
    
    // STEP 3: Capture pre-state
    uint256 balanceBefore = balanceOf(e.msg.sender);
    uint256 recipientBefore = balanceOf(recipient);
    uint256 totalBefore = totalSupply();
    
    // STEP 4: Execute action (reverts pruned — verified in companion rule)
    transfer(e, recipient, amount);
    
    // STEP 5: Assert post-conditions
    assert balanceOf(e.msg.sender) == balanceBefore - amount,
        "Sender balance not decreased correctly";
    assert balanceOf(recipient) == recipientBefore + amount,
        "Recipient balance not increased correctly";
    assert totalSupply() == totalBefore,
        "Total supply changed during transfer";
}
```

#### Pattern B: Complete Rule with Revert Verification (PREFERRED)

```cvl
/**
 * @title [RULE_NAME] — Complete (Liveness + Effect + No Side Effect)
 * @notice Verifies BOTH success and failure behavior
 * @dev Source: [security property document reference]
 *      Depends on invariants: [list]
 */
rule rule_[name]_complete(env e, [parameters]) {
    // STEP 1: Load valid state
    validState();
    validEnv(e);
    
    // STEP 2: Capture pre-state
    uint256 balanceBefore = balanceOf(e.msg.sender);
    uint256 recipientBefore = balanceOf(recipient);
    uint256 totalBefore = totalSupply();
    
    // STEP 3: Execute with revert tracking
    transfer@withrevert(e, recipient, amount);
    bool success = !lastReverted;
    
    // STEP 4: LIVENESS — enumerate ALL revert conditions
    assert success <=> (
        e.msg.sender != 0 &&
        recipient != 0 &&
        balanceBefore >= to_mathint(amount)
    );
    
    // STEP 5: EFFECT — correct state changes on success
    assert success => to_mathint(balanceOf(e.msg.sender)) == balanceBefore - to_mathint(amount);
    assert success => to_mathint(balanceOf(recipient)) == recipientBefore + to_mathint(amount);
    assert success => to_mathint(totalSupply()) == totalBefore;
}
```

> **Why Pattern B is preferred:** It proves the function reverts **if and only if** the listed
> conditions hold. If you miss a revert condition, the biconditional `<=>` fails. If the
> function doesn't revert when it should, the biconditional also fails. This is the
> industry-standard approach used in OpenZeppelin verification.

#### Pattern C: Dedicated Revert Rule (companion to Pattern A)

```cvl
/**
 * @title [RULE_NAME] — Revert Conditions
 * @notice Proves that the function reverts if and only if specific conditions hold
 */
rule rule_[name]_revert(env e, [parameters]) {
    // Capture all pre-state needed for revert analysis
    uint256 balance = balanceOf(e.msg.sender);
    
    transfer@withrevert(e, recipient, amount);
    
    // Exhaustive revert enumeration
    assert lastReverted <=> (
        e.msg.value != 0 ||             // non-payable
        e.msg.sender == 0 ||            // zero-address sender
        recipient == 0 ||               // zero-address recipient
        balance < to_mathint(amount)    // insufficient balance
    );
}
```
```

### Parametric Rule Pattern (For "Any Function" Properties)

```cvl
/**
 * @title [PARAMETRIC_RULE_NAME]
 * @notice Verifies property holds across all contract functions
 */
rule parametric_[name](method f) 
    filtered { 
        // Exclude irrelevant or trusted functions
        f -> f.contract == currentContract &&
             f.selector != sig:trustedAdmin().selector &&
             f.selector != sig:irrelevantView().selector
    }
{
    // Load valid state
    validState();
    
    // Capture state before ANY function
    uint256 valueBefore = someValue();
    bool conditionBefore = someCondition();
    
    // Execute any function
    env e;
    require e.msg.sender != currentContract;
    require e.block.timestamp <= max_uint40;
    
    calldataarg args;
    f(e, args);
    
    // Verify property preserved
    assert conditionBefore => someCondition(),
        "Property violated by function";
}
```

### State Transition Rule Pattern

```cvl
/**
 * @title State Transition: [FROM] → [TO]
 * @notice Only [function] can transition from [FROM] to [TO]
 */
rule stateTransition_[from]_to_[to](method f) 
    filtered { f -> f.contract == currentContract }
{
    validState();
    
    // Assume we start in FROM state
    require getState() == FROM_STATE;
    
    env e;
    validEnv(e);
    calldataarg args;
    f(e, args);
    
    // If we end in TO state, verify it was the right function
    assert getState() == TO_STATE => (
        f.selector == sig:allowedTransitionFunction().selector
    ), "Invalid state transition to [TO]";
}

/**
 * @title State Finality: [TERMINAL_STATE] is final
 * @notice Once in [TERMINAL_STATE], no function can change it
 */
rule stateFinal_[terminal_state](method f) 
    filtered { f -> f.contract == currentContract }
{
    validState();
    
    bool inTerminalBefore = getState() == TERMINAL_STATE;
    
    env e;
    validEnv(e);
    calldataarg args;
    f(e, args);
    
    assert inTerminalBefore => getState() == TERMINAL_STATE,
        "[TERMINAL_STATE] must be final";
}
```

### Access Control Rule Pattern

```cvl
/**
 * @title Only [ROLE] can call [FUNCTION]
 * @notice Access control verification
 */
rule accessControl_[function]_onlyRole(env e, [parameters]) {
    validState();
    validEnv(e);
    
    // Capture who the authorized role is
    address authorizedRole = getRoleAddress();
    
    // Attempt the restricted function
    restrictedFunction@withrevert(e, [parameters]);
    
    // If it succeeded, caller must be authorized
    assert !lastReverted => e.msg.sender == authorizedRole,
        "Unauthorized caller executed restricted function";
}
```

---

## Phase 4: Final Validation Checklist

Before considering any spec complete, verify ALL items:

### Execution Closure Checklist

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | All external contracts enumerated | Phase -2 |
| ☐ | All external functions have modeling strategy | Phase -2 |
| ☐ | Methods block includes all external calls | Phase -1 |
| ☐ | Every NONDET has documented justification | Phase -1 |
| ☐ | All contract addresses are bound or constrained | Address Closure Rule |
| ☐ | All read functions paired with write modeling | Read/Write Pairing |
| ☐ | Custom summaries for complex external behavior | Phase -1 |

### Causal Closure Checklist

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | **`satisfy` reachability rules PASS for all entry points** | Proves no function always reverts (anti-vacuity) ← NEW v1.8 |
| ☐ | Mutation paths enumerated for all property variables | Phase 0 |
| ☐ | Constructor effects modeled (init_state axioms) | Phase 0 |
| ☐ | Ghosts have Sstore hooks for ALL mutation paths | Phase 1 |
| ☐ | Ghosts have Sload hooks enforcing relationships | Phase 1 |
| ☐ | Mutation path validation rules pass | Phase 0 |
| ☐ | Ghost synchronization validation passes | Phase 0 |

### Bounded State Checklist

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | Array lengths bounded realistically (< 1000) | validState() |
| ☐ | Timestamps bounded (<= max_uint40) | validEnv() |
| ☐ | IDs/counters bounded realistically | validState() |
| ☐ | Loop iterations bounded in preserved blocks | Invariants |

### Property Implementation Checklist

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | Property correctly classified (INVARIANT/RULE) | Phase 2 |
| ☐ | validState() called at rule start | Phase 3 |
| ☐ | validEnv(e) called when using env | Phase 3 |
| ☐ | requireInvariant for all dependencies | Preserved blocks |
| ☐ | filtered clause excludes irrelevant functions | Invariants/Rules |
| ☐ | e.msg.sender != currentContract constraint | Unless modeling reentrancy |
| ☐ | No require for invariants (use requireInvariant) | Phase 3 |
| ☐ | No assume for safety properties | Never |

### Revert/Failure-Path Checklist (NEW in v1.6)

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | Every state-changing function has `@withrevert` coverage | At minimum in a dedicated revert rule |
| ☐ | Liveness assertions use `<=>` (not just `=>`) | Biconditional ensures exhaustive revert enumeration |
| ☐ | `lastReverted` captured immediately after each `@withrevert` call | Prevents overwriting by subsequent calls |
| ☐ | Non-payable revert (`e.msg.value != 0`) included in liveness conditions | The Prover WILL find this |
| ☐ | Access control reverts verified (unauthorized caller → revert) | Not just "authorized caller → success" |
| ☐ | Overflow/underflow reverts verified where relevant | Especially for unchecked blocks |
| ☐ | `use builtin rule uncheckedOverflow` run on contracts with unchecked blocks | Auto-catches unsafe unchecked math (v8.8.0+) |
| ☐ | `use builtin rule safeCasting` run on contracts with explicit casts | Auto-catches out-of-bounds casts (v8.8.0+) |
| ☐ | No rule relies solely on default revert pruning | If reverts are expected, they must be explicitly tested |

### Quality Checklist

| ✅ | Item | Notes |
|----|------|-------|
| ☐ | Every invariant has @title, @notice, @dev | Documentation |
| ☐ | Dependencies documented in @dev | Traceability |
| ☐ | Source property document referenced | Traceability |
| ☐ | Counterexamples (if any) are reachable on-chain | Validation |
| ☐ | No HAVOC-based counterexamples | Modeling complete |

---

## CVL 2.0 Templates

### Complete Spec File Template

```cvl
/**
 * ================================================================
 * CERTORA SPECIFICATION — [CONTRACT_NAME]
 * ================================================================
 * See header documentation requirements above
 */

// ═══════════════════════════════════════════════════════════════
// IMPORTS
// ═══════════════════════════════════════════════════════════════
import "Common.spec";

// ═══════════════════════════════════════════════════════════════
// USING DECLARATIONS
// ═══════════════════════════════════════════════════════════════
using ContractA as contractA;
using ContractB as contractB;

// ═══════════════════════════════════════════════════════════════
// METHODS BLOCK
// ═══════════════════════════════════════════════════════════════
methods {
    // Envfree declarations
    function myContract.viewFunction() external returns (uint256) envfree;
    
    // Dispatcher summaries
    function _.dispatchedFunction() external => DISPATCHER(true);
    
    // NONDET summaries (justified)
    // JUSTIFICATION: [reason]
    function _.externalFunction() external => NONDET;
    
    // Custom summaries
    function Complex.function(bytes) external returns (bytes) 
        => customSummary();
}

// ═══════════════════════════════════════════════════════════════
// DEFINITIONS (Reusable CVL Expressions — NEW in v1.5)
// ═══════════════════════════════════════════════════════════════
definition nonpayable(env e) returns bool = e.msg.value == 0;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
definition balanceLimited(address a) returns bool = balanceOf(a) < max_uint256;

// ═══════════════════════════════════════════════════════════════
// CUSTOM SUMMARY IMPLEMENTATIONS
// ═══════════════════════════════════════════════════════════════
function customSummary() returns bytes {
    bytes b;
    return b;
}

// ═══════════════════════════════════════════════════════════════
// GHOST DECLARATIONS
// ═══════════════════════════════════════════════════════════════
ghost mathint sumVariable {
    init_state axiom sumVariable == 0;
}

// ═══════════════════════════════════════════════════════════════
// HOOKS
// ═══════════════════════════════════════════════════════════════
hook Sstore mapping[KEY address k] uint256 newVal (uint256 oldVal) {
    sumVariable = sumVariable + newVal - oldVal;
}

hook Sload uint256 val mapping[KEY address k] {
    require val <= sumVariable;
}

// ═══════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ═══════════════════════════════════════════════════════════════
function validState() {
    // Bounds
    require someArray.length < 1000;
    
    // Invariant chain
    requireInvariant invariant1();
    requireInvariant invariant2();
}

function validEnv(env e) {
    require e.block.timestamp <= max_uint40;
    require e.msg.sender != currentContract;
}

// ═══════════════════════════════════════════════════════════════
// INVARIANTS (ordered by dependency level — see DAG below)
// ═══════════════════════════════════════════════════════════════
//
// INVARIANT DEPENDENCY DAG (NEW v1.9 — Cycle Detection):
//   Level 1: invariant1  (no dependencies — prove FIRST with --rule)
//       ↓
//   Level 2: invariant2  (depends on invariant1)
//       ↓
//   Level 3: ...         (depends on Level 2)
//
// 🚨 CYCLE DETECTION PROTOCOL:
//   1. Every invariant MUST have @dev Level: N annotation
//   2. requireInvariant may ONLY reference invariants at a LOWER level
//   3. Prove Level 1 invariants in isolation: --rule "invariant1"
//   4. Only after Level 1 passes, prove Level 2, etc.
//   5. If you cannot assign a level → you have a circular dependency → STOP and refactor
//   6. Document the DAG in causal_validation.md
// ═══════════════════════════════════════════════════════════════

/// @title Level 1 invariant (no dependencies)
/// @dev Level: 1 | Dependencies: NONE | Prove first with: --rule "invariant1"
invariant invariant1()
    condition1
    filtered { f -> f.contract == currentContract }
    {
        preserved with (env e) {
            require e.msg.sender != currentContract;
        }
    }

/// @title Level 2 invariant (depends on Level 1)
/// @dev Level: 2 | Dependencies: invariant1 | Prove after Level 1 passes
invariant invariant2()
    condition2
    filtered { f -> f.contract == currentContract }
    {
        preserved with (env e) {
            requireInvariant invariant1();
            require e.msg.sender != currentContract;
        }
    }

// ═══════════════════════════════════════════════════════════════
// RULES
// ═══════════════════════════════════════════════════════════════

/// @title Behavioral rule
rule myRule(env e, address user, uint256 amount) {
    validState();
    validEnv(e);
    
    uint256 before = getValue(user);
    
    myFunction(e, user, amount);
    
    assert getValue(user) == before + amount,
        "Value not updated correctly";
}

/// @title Liveness / Effect / No-Side-Effect Rule (NEW in v1.5)
/// Industry-standard pattern from OpenZeppelin specifications.
/// See cvl-language-deep-dive.md Section 15 and verification-playbooks.md
rule livenessEffectTemplate(env e) {
    require nonpayable(e);
    require nonzerosender(e);
    
    // Pre-state snapshot
    uint256 valueBefore = getValue(e.msg.sender);
    address otherAccount;
    uint256 otherValueBefore = getValue(otherAccount);
    
    // Action
    myFunction@withrevert(e, amount);
    bool success = !lastReverted;
    
    // LIVENESS: success <=> preconditions
    assert success <=> (
        valueBefore >= to_mathint(amount)
    );
    
    // EFFECT: success => correct state changes
    assert success => to_mathint(getValue(e.msg.sender)) == valueBefore - to_mathint(amount);
    
    // NO SIDE EFFECT: uninvolved accounts unchanged
    assert getValue(otherAccount) != otherValueBefore => otherAccount == e.msg.sender;
}

/// @title Parametric rule
rule parametricRule(method f) 
    filtered { f -> f.contract == currentContract }
{
    validState();
    
    bool conditionBefore = someCondition();
    
    env e;
    validEnv(e);
    calldataarg args;
    f(e, args);
    
    assert conditionBefore => someCondition(),
        "Condition not preserved";
}
```

---

## How to Use This Framework

### For Any New Project:

```
1. PHASE -2: SCOPE DEFINITION
   - Create contract enumeration table
   - Create function enumeration table
   - Document address bindings

2. PHASE -1: EXECUTION SURFACE CLOSURE  
   - Write methods block with all summaries
   - Document NONDET justifications
   - Implement custom summaries

3. PHASE 0: CAUSAL CLOSURE VALIDATION
   - Create property analysis documents
   - Write and run validation rules
   - Fix any causal gaps before proceeding

4. PHASE 1: FOUNDATION LAYER
   - Define ghosts with init_state axioms
   - Write Sstore and Sload hooks
   - Create validState() function
   - Write and prove foundation invariants

5. PHASE 2: PROPERTY CLASSIFICATION
   - Use decision tree for each property
   - Document classification reasoning

6. PHASE 3: PROPERTY IMPLEMENTATION
   - Follow appropriate pattern (invariant/rule)
   - Include all required elements
   - Chain dependencies correctly

7. PHASE 4: FINAL VALIDATION
   - Complete all checklists
   - Verify no HAVOC counterexamples
   - Ensure all counterexamples are reachable
```

### If You Get Counterexamples:

**For comprehensive CE debugging, see:**
- **certora-ce-diagnosis-framework.md** - Systematic 5-phase diagnosis
- **best-practices-from-certora.md Section 2** - Tutorial-based investigation workflow

| Counterexample Type | Root Cause | Fix |
|---------------------|------------|-----|
| Balance > totalSupply | Missing sum tracking | Add ghost + hooks |
| Impossible timestamp | Missing bound | Add `require e.block.timestamp <= max_uint40` |
| Self-call | Missing constraint | Add `require e.msg.sender != currentContract` |
| HAVOC values | Unmodeled external call | Add summary in methods block |
| Uninitialized state | Missing init_state axiom | Add axiom to ghost |
| Dependency failure | Missing requireInvariant | Add to preserved block |

---

> **Remember:** The prover is always correct. If it finds a counterexample, either:
> 1. You found a real bug, or  
> 2. Your model is incomplete
>
> This framework ensures #2 happens as rarely as possible.
