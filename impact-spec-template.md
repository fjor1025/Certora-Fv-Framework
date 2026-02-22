# Economic Impact Tracking Specification Template

> **Framework Version:** v3.0 (Offensive Verification)  
> **Purpose:** First-class primitives for tracking attacker profit, value extraction, and system impact  
> **Usage:** Import or copy into every audit specification

---

## Overview

This template provides the **missing economic primitives** for exploit discovery. Traditional FV asks "does this match the spec?" — this template asks "can someone profit from breaking this?"

### Core Concepts

| Concept | Definition | CVL Representation |
|---------|------------|-------------------|
| **Attacker Value** | Total assets controlled by potential attacker | `persistent ghost mapping(...) actor_value` |
| **System Value** | Total assets held by protocol | `persistent ghost mathint total_system_value` |
| **Value Extraction** | Assets moving from protocol to attacker | `persistent ghost mathint total_value_extracted` |
| **Irreversible Loss** | Extraction that cannot be recovered | `persistent ghost bool irreversible_loss_occurred` |

---

## Complete Impact Tracking Specification

```cvl
/**
 * ================================================================
 * ECONOMIC IMPACT TRACKING INFRASTRUCTURE
 * ================================================================
 * 
 * Purpose: Track value flows to detect profitable attack paths
 * 
 * Usage:
 *   1. Include this spec via `use spec "impact.spec";` OR
 *   2. Copy relevant sections into your main spec
 *   3. Run anti-invariants EXPECTING them to fail
 *   4. If they pass = no exploit found; if they fail = EXPLOIT FOUND
 *
 * Philosophy:
 *   - Traditional specs ask: "Does code match intent?"
 *   - This spec asks: "Can an attacker profit?"
 *   
 *   We write rules that SHOULD fail. Finding a counterexample = finding an exploit.
 * ================================================================
 */

// ================================================================
// SECTION 1: ACTOR VALUE TRACKING
// ================================================================

/**
 * @title Per-Actor Value Tracking
 * @notice Tracks the total value controlled by each address
 * @dev Updates via hooks on all value-bearing state changes
 */
persistent ghost mapping(address => mathint) actor_value {
    // All actors start with 0 value we're tracking
    // (Their external holdings are not our concern)
    init_state axiom forall address a. actor_value[a] == 0;
}

/**
 * @title Total System Value
 * @notice Sum of all value held by the protocol contracts
 * @dev Must equal sum of all actor claims if system is solvent
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost mathint total_system_value {
    init_state axiom total_system_value == 0;
}

/**
 * @title Value Extraction Counter
 * @notice Cumulative value that has left the protocol to external addresses
 * @dev Only increases; represents potential attacker profit
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost mathint total_value_extracted {
    init_state axiom total_value_extracted == 0;
}

// ================================================================
// SECTION 2: IMPACT CATEGORY FLAGS
// ================================================================

/**
 * @title Insolvency Flag
 * @notice True if protocol obligations exceed holdings
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost bool insolvent_state {
    init_state axiom insolvent_state == false;
}

/**
 * @title Share Dilution Factor
 * @notice Tracks unexpected inflation of claims/shares
 * @dev Non-zero value indicates dilution attack possible
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost mathint dilution_factor {
    init_state axiom dilution_factor == 0;
}

/**
 * @title Debt Socialization Tracking
 * @notice Tracks losses pushed onto innocent users
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost mapping(address => mathint) socialized_loss {
    init_state axiom forall address a. socialized_loss[a] == 0;
}

/**
 * @title Liquidity Freeze Flag
 * @notice True if legitimate users are blocked from withdrawing
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost bool liquidity_frozen {
    init_state axiom liquidity_frozen == false;
}

/**
 * @title Irreversible Loss Flag
 * @notice True if extraction cannot be recovered by any mechanism
 * @dev MUST be persistent — survives HAVOC from external calls
 */
persistent ghost bool irreversible_loss_occurred {
    init_state axiom irreversible_loss_occurred == false;
}

// ================================================================
// SECTION 3: VALUE FLOW HOOKS
// ================================================================

/**
 * @title ERC20 Balance Hook
 * @notice Tracks token balance changes per actor
 * @dev Adapt selector to match your token's storage layout
 */
hook Sstore _balances[KEY address account] uint256 newBalance (uint256 oldBalance) {
    mathint delta = to_mathint(newBalance) - to_mathint(oldBalance);
    actor_value[account] = actor_value[account] + delta;
}

/**
 * @title ETH Transfer Hook
 * @notice Tracks native ETH flows between addresses
 * @dev Captures CALL opcode with value transfer
 */
hook CALL(uint g, address target, uint value, uint argsOffset, uint argsLength, uint retOffset, uint retLength) uint retval {
    if (value > 0) {
        // ETH leaving current contract
        actor_value[currentContract] = actor_value[currentContract] - to_mathint(value);
        // ETH arriving at target
        actor_value[target] = actor_value[target] + to_mathint(value);
        
        // Track extraction if target is not a protocol contract
        // (You must define isProtocolContract for your specific protocol)
        // if (!isProtocolContract(target)) {
        //     total_value_extracted = total_value_extracted + to_mathint(value);
        // }
    }
}

/**
 * @title Share/Claim Balance Hook
 * @notice Tracks share token or claim balance changes
 * @dev Adapt to your protocol's share tracking storage
 */
// hook Sstore shares[KEY address account] uint256 newShares (uint256 oldShares) {
//     mathint delta = to_mathint(newShares) - to_mathint(oldShares);
//     actor_value[account] = actor_value[account] + delta * share_price();
// }

// ================================================================
// SECTION 4: HELPER DEFINITIONS
// ================================================================

/**
 * @title Profitable Attack Detection
 * @notice Returns true if actor gained value from the operation
 */
definition profitable_attack(address attacker, mathint value_before, mathint value_after) returns bool =
    value_after > value_before;

/**
 * @title Harmful Extraction Detection
 * @notice Returns true if system lost value
 */
definition harmful_extraction(mathint system_before, mathint system_after) returns bool =
    system_after < system_before;

/**
 * @title Net Position Change
 * @notice Calculates how much an actor's position changed
 */
definition position_delta(address actor, mathint before, mathint after) returns mathint =
    after - before;

/**
 * @title Non-payable Precondition
 * @notice Standard filter for non-payable functions
 */
definition nonpayable(env e) returns bool = e.msg.value == 0;

/**
 * @title Valid Sender Precondition
 * @notice Excludes zero address as sender
 */
definition valid_sender(env e) returns bool = e.msg.sender != 0;

// ================================================================
// SECTION 5: ANTI-INVARIANTS (Expected to FAIL = Bug Found)
// ================================================================

/**
 * @title Attacker Profit Search
 * @notice Searches for any function that allows caller to profit
 * @dev EXPECTED TO FAIL if exploit exists
 * 
 * If this rule is VIOLATED, the counterexample contains:
 *   - Function called
 *   - Parameters used
 *   - Attacker profit amount
 */
rule attacker_cannot_profit(env e, method f) 
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);
    require nonpayable(e);  // Remove if testing payable functions
    
    address attacker = e.msg.sender;
    mathint value_before = actor_value[attacker];
    
    calldataarg args;
    f(e, args);
    
    mathint value_after = actor_value[attacker];
    
    // THIS SHOULD FAIL IF THERE'S A PROFIT PATH
    assert !profitable_attack(attacker, value_before, value_after), 
        "EXPLOIT FOUND: Caller profited from this call";
}

/**
 * @title System Value Conservation
 * @notice Verifies protocol value doesn't disappear
 * @dev EXPECTED TO FAIL if value can be extracted
 */
rule system_value_conserved(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);
    
    mathint system_before = total_system_value;
    
    calldataarg args;
    f@withrevert(e, args);
    
    mathint system_after = total_system_value;
    
    // Value should not appear or disappear
    assert system_after == system_before, 
        "EXPLOIT FOUND: System value changed unexpectedly";
}

/**
 * @title Zero-Sum Verification
 * @notice If one actor gains, another must lose equivalently
 * @dev Detects value creation from nothing
 */
rule zero_sum_transfers(env e, method f, address other)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);
    require e.msg.sender != other;
    
    address caller = e.msg.sender;
    
    mathint caller_before = actor_value[caller];
    mathint other_before = actor_value[other];
    mathint system_before = total_system_value;
    
    calldataarg args;
    f(e, args);
    
    mathint caller_after = actor_value[caller];
    mathint other_after = actor_value[other];
    mathint system_after = total_system_value;
    
    mathint caller_delta = caller_after - caller_before;
    mathint other_delta = other_after - other_before;
    mathint system_delta = system_after - system_before;
    
    // All value movements must net to zero
    assert caller_delta + other_delta + system_delta == 0,
        "EXPLOIT FOUND: Value created or destroyed";
}

// ================================================================
// SECTION 6: IMPACT INVARIANTS
// ================================================================

/**
 * @title No Value Extraction Invariant
 * @notice Protocol should never have value extracted
 */
invariant no_value_extraction()
    total_value_extracted == 0
{
    preserved {
        // Add requireInvariant calls for dependencies
    }
}

/**
 * @title Solvency Invariant
 * @notice Protocol must always be solvent
 */
invariant always_solvent()
    !insolvent_state
{
    preserved {
        // Add requireInvariant calls for dependencies
    }
}

/**
 * @title No Share Dilution Invariant
 * @notice Share supply should not be inflatable
 */
invariant no_share_dilution()
    dilution_factor == 0
{
    preserved {
        // Add requireInvariant calls for dependencies
    }
}

/**
 * @title No Liquidity Freeze Invariant
 * @notice Users must always be able to exit
 */
invariant no_liquidity_freeze()
    !liquidity_frozen
{
    preserved {
        // Add requireInvariant calls for dependencies
    }
}

// ================================================================
// SECTION 7: SATISFY-BASED ATTACK SEARCH
// ================================================================

/**
 * @title Find Profit Path
 * @notice Actively searches for profitable attack parameters
 * @dev Uses satisfy to find inputs that maximize profit
 */
rule find_profitable_inputs(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);
    
    address attacker = e.msg.sender;
    mathint value_before = actor_value[attacker];
    
    calldataarg args;
    f@withrevert(e, args);
    
    mathint value_after = actor_value[attacker];
    mathint profit = value_after - value_before;
    
    // FIND inputs that create profit
    // If this VERIFIES, exploit parameters are in the witness
    satisfy profit > 0;
}

/**
 * @title Find Insolvency Path
 * @notice Searches for inputs that cause insolvency
 */
rule find_insolvency_path(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);
    require !insolvent_state;  // Start solvent
    
    calldataarg args;
    f@withrevert(e, args);
    
    // FIND inputs that cause insolvency
    satisfy insolvent_state;
}

/**
 * @title Iterative Threshold Profit Search
 * @notice Finds approximate MAXIMUM profit via binary search thresholds
 * @dev Run repeatedly with increasing thresholds:
 *      satisfy profit > 0       → SAT? Continue
 *      satisfy profit > 10^6    → SAT? Continue
 *      satisfy profit > 10^9    → SAT? Continue
 *      satisfy profit > 10^12   → UNSAT → Max exploit ≈ 10^9–10^12 range
 *
 * IMPORTANT: Certora's SMT solver finds ANY satisfying assignment, not the
 * maximum. This iterative tightening protocol provides a workaround.
 */
rule find_max_profit_threshold(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);

    address attacker = e.msg.sender;
    mathint value_before = actor_value[attacker];

    calldataarg args;
    f@withrevert(e, args);

    mathint value_after = actor_value[attacker];
    mathint profit = value_after - value_before;

    // ════════════════════════════════════════════════════════════════
    // ITERATIVE THRESHOLD PROTOCOL
    // ════════════════════════════════════════════════════════════════
    // Un-comment ONE threshold at a time, run, and advance if SAT:
    //
    // Round 1: satisfy profit > 0;
    // Round 2: satisfy profit > 1000;              // > 1K wei
    // Round 3: satisfy profit > 1000000;           // > 1M wei
    // Round 4: satisfy profit > 1000000000;        // > 1 Gwei
    // Round 5: satisfy profit > 10^15;             // > 0.001 ETH
    // Round 6: satisfy profit > 10^18;             // > 1 ETH
    // Round 7: satisfy profit > 1000 * 10^18;      // > 1K ETH
    //
    // When a round returns UNSAT, the maximum exploitable profit
    // lies between the previous and current threshold.
    // ════════════════════════════════════════════════════════════════
    satisfy profit > 0;
}
```

---

```cvl
// ================================================================
// SECTION 8: ANTI-INVARIANT LIVENESS CHECKS
// ================================================================
//
// PURPOSE: Prove that impact hooks are ALIVE for each function.
//
// A passing anti-invariant (VERIFIED attacker_cannot_profit) could mean:
//   1. Protocol is genuinely safe (ideal)
//   2. Hooks are INCOMPLETE — value flow not captured (DANGEROUS)
//   3. require() constraints filter out the attack (DANGEROUS)
//
// These liveness rules use `satisfy` to confirm that the hooks
// actually observe value movement for each state-changing function.
// If a liveness rule is VIOLATED (no satisfying assignment), it means
// the hooks DO NOT capture that function's value effects — and
// `attacker_cannot_profit` is VACUOUS for that function.
//
// HOW TO USE:
//   1. Copy this section into your spec.
//   2. Replace TARGET_FUNCTION with each state-changing function.
//   3. Run each liveness rule. VIOLATED = hooks are blind to that function.
//   4. Fix hooks BEFORE trusting any anti-invariant result.
// ================================================================

/**
 * @title Hook Liveness Check (Parametric)
 * @notice Confirms that actor_value hooks observe value changes
 * @dev If VIOLATED: hooks are blind to this function's value effects.
 *      Fix hooks before trusting attacker_cannot_profit.
 */
rule impact_hook_liveness(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);

    address actor = e.msg.sender;
    mathint before = actor_value[actor];

    calldataarg args;
    f@withrevert(e, args);

    mathint after = actor_value[actor];

    // If this satisfy is VIOLATED, the hooks don't see
    // this function's value effects — anti-invariants are vacuous.
    satisfy before != after,
        "Hooks capture value change for this function";
}

/**
 * @title System Value Hook Liveness
 * @notice Confirms that total_system_value is updated by functions
 * @dev If VIOLATED for a function that should move protocol value,
 *      system_value_conserved results are unreliable.
 */
rule system_value_hook_liveness(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);

    mathint sys_before = total_system_value;

    calldataarg args;
    f@withrevert(e, args);

    mathint sys_after = total_system_value;

    satisfy sys_before != sys_after,
        "Hooks capture system value change for this function";
}
```

---

## Value Flow Completeness Checklist

> **MANDATORY before trusting any anti-invariant result.**
>
> Cross-reference every asset identified in Phase 0 Asset Flow Trace against your hooks.
> If any row is uncovered, `attacker_cannot_profit` may return false VERIFIED.

| Value Channel | Storage / Mechanism | Hook Required | Covered? |
|---------------|---------------------|---------------|----------|
| ERC-20 token balances | `_balances[address]` | `hook Sstore _balances[KEY address]` | [ ] |
| Native ETH via CALL | CALL opcode with `value > 0` | `hook CALL(...)` | [ ] |
| Native ETH via SELFDESTRUCT | SELFDESTRUCT opcode | Manual modeling required | [ ] |
| Native ETH via CREATE2 | CREATE2 with value | Manual modeling required | [ ] |
| Share/vault tokens (ERC-4626) | `shares[address]`, `_shares[address]` | `hook Sstore shares[KEY address]` | [ ] |
| LP token balances | LP `_balances[address]` | Separate hook per LP token | [ ] |
| Multi-token (ERC-1155) | `_balances[id][address]` | `hook Sstore _balances[KEY uint256][KEY address]` | [ ] |
| Internal accounting | Protocol-specific mappings | Custom hooks per mapping | [ ] |
| Rebasing token balances | Implicit via supply changes | Track supply + share price | [ ] |
| Fee accruals | Fee storage variables | `hook Sstore _accumulatedFees` etc. | [ ] |

**Liveness Verification:**
- [ ] Ran `impact_hook_liveness` for ALL state-changing functions
- [ ] Ran `system_value_hook_liveness` for value-moving functions
- [ ] Every function that moves value shows `satisfy` SAT (hooks are alive)
- [ ] Functions where `satisfy` is VIOLATED have been investigated and hooks fixed

---

## Usage Guide

### Step 1: Adapt Hooks to Your Protocol

The template provides generic hooks. You must adapt them to your protocol's storage layout:

```cvl
// Example: Adapt balance hook for your token
hook Sstore _balances[KEY address account] uint256 newBalance (uint256 oldBalance) {
    // Change _balances to your actual storage variable name
    // May need to check contract instance: currentContract == token
}
```

### Step 2: Define Protocol Boundaries

Specify which addresses are protocol contracts vs external:

```cvl
definition isProtocolContract(address a) returns bool =
    a == vault ||
    a == staking ||
    a == governance;
```

### Step 3: Run Anti-Invariants

```bash
# Run the profit search rule
certoraRun config.conf --rule attacker_cannot_profit

# If VIOLATED: EXPLOIT FOUND - examine counterexample
# If VERIFIED: No single-tx profit path (run multi-step next)
```

### Step 4: Convert Counterexamples to PoCs

When a rule fails, extract from the CE:

| CE Element | Exploit Element |
|------------|-----------------|
| Function called | Attack entry point |
| msg.sender | Attacker address |
| Arguments | Attack parameters |
| Storage before/after | Expected state changes |
| actor_value delta | Profit amount |

---

## Integration Checklist

- [ ] Copied impact tracking ghosts to my spec (ALL must be `persistent`)
- [ ] Adapted hooks to my protocol's storage layout
- [ ] Defined `isProtocolContract` for my protocol
- [ ] Completed Value Flow Completeness Checklist (all assets covered)
- [ ] Added value flow tracking for all assets (tokens, ETH, shares)
- [ ] Ran `impact_hook_liveness` for every state-changing function ← **NEW**
- [ ] Ran `system_value_hook_liveness` for value-moving functions ← **NEW**
- [ ] Fixed any hooks where liveness `satisfy` was VIOLATED ← **NEW**
- [ ] Ran `attacker_cannot_profit` rule
- [ ] Ran `system_value_conserved` rule
- [ ] Ran `find_profitable_inputs` satisfy rule
- [ ] Ran `find_max_profit_threshold` with iterative tightening ← **NEW**
- [ ] Converted any CEs to Foundry PoCs
