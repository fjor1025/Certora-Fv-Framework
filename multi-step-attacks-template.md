# Multi-Step Attack Patterns Template

> **Framework Version:** v3.1 (Adversarial Verification Loop)  
> **Purpose:** Model stateful attackers across transaction boundaries  
> **Usage:** Copy patterns for flash loan, sandwich, and staged attacks

---

## Overview

Real DeFi exploits are rarely single-transaction. This template provides patterns for:

| Attack Class | Transactions | Example |
|--------------|--------------|---------|
| **Flash Loan** | 1 (internal multi-step) | Borrow → Manipulate → Extract → Repay |
| **Sandwich** | 3 (atomic block) | Frontrun → Victim → Backrun |
| **Staged** | N (across blocks) | Accumulate → Trigger → Extract |
| **Governance** | 2+ (with delay) | Propose → Wait → Execute → Extract |

---

## CVL State Model: How Multi-Step Attacks Work

> **Critical Understanding:** In CVL, sequential function calls within a rule
> operate on **evolved state**. When you write:
> ```cvl
> f1(e1, args1);
> f2(e2, args2);
> f3(e3, args3);
> ```
> After `f1` executes, ALL storage is updated. `f2` sees the post-`f1` state.
> `f3` sees the post-`f2` state. This is how the Prover models sequential
> blockchain transactions natively.

### State Carry-Over Guarantee

| State Type | Carries Over? | Mechanism |
|------------|---------------|----------|
| Contract storage (slots) | ✅ Yes | Prover models EVM storage automatically |
| `ghost` variables | ✅ Yes (if no external calls) | Updated by hooks, reverted on revert |
| `persistent ghost` variables | ✅ **Always** | Survives HAVOC and reverts |
| `env` variables between calls | ❌ Independent | Each `env` is a free variable; constrain via `require` |
| `calldataarg` between calls | ❌ Independent | Each `calldataarg` is a free variable |

### Why Persistent Ghosts Matter for Multi-Step Attacks

If Step 1 calls an **external contract** (e.g., flash loan provider), the Prover
cannot see its implementation. Without `persistent`:

```
Step 1: f1 calls external → HAVOC → ghost resets to arbitrary value
Step 2: f2 reads ghost → sees garbage, not Step 1's delta
```

With `persistent`:

```
Step 1: f1 calls external → ghost SURVIVES HAVOC
Step 2: f2 reads ghost → sees Step 1's actual delta
```

This is why **every ghost in this template is `persistent`** — multi-step
attacks inherently involve cross-contract calls.

### Intermediate State Snapshots

To capture state between steps for comparison, use local variables:

```cvl
// Capture post-step-1 state
mathint after_step1 = actor_value[attacker];

f2(e2, args2);

// Compare: what did step 2 change?
mathint step2_delta = actor_value[attacker] - after_step1;
```

The Prover tracks these automatically — no explicit snapshot mechanism needed.

---

## Flash Loan Attack Pattern

Flash loans enable atomic multi-step attacks within a single transaction.

```cvl
/**
 * ================================================================
 * FLASH LOAN ATTACK MODELING
 * ================================================================
 * 
 * Flash loan attack structure:
 *   1. Borrow large amount (no collateral required)
 *   2. Use borrowed funds to manipulate protocol state
 *   3. Extract value from manipulated state
 *   4. Repay flash loan + fee
 *   5. Keep profit
 *
 * Key insight: Attacker has temporary access to unlimited capital
 * ================================================================
 */

// Track flash loan state
persistent ghost mathint flash_borrowed {
    init_state axiom flash_borrowed == 0;
}

persistent ghost mathint flash_profit {
    init_state axiom flash_profit == 0;
}

persistent ghost bool in_flash_loan {
    init_state axiom in_flash_loan == false;
}

/**
 * @title Flash Loan Attack Search
 * @notice Simulates flash loan attack pattern
 * @dev Models: borrow → [arbitrary operations] → repay → profit check
 */
rule flash_loan_attack_search(env e_borrow, env e_attack, env e_repay) {
    // Same attacker controls all steps
    address attacker = e_borrow.msg.sender;
    require e_attack.msg.sender == attacker;
    require e_repay.msg.sender == attacker;
    
    // Same block (atomic)
    require e_borrow.block.number == e_attack.block.number;
    require e_attack.block.number == e_repay.block.number;
    
    // Same timestamp
    require e_borrow.block.timestamp == e_attack.block.timestamp;
    require e_attack.block.timestamp == e_repay.block.timestamp;
    
    // Capture initial state
    mathint attacker_value_initial = actor_value[attacker];
    mathint protocol_value_initial = total_system_value;
    
    // ========================================
    // STEP 1: Flash Loan Borrow
    // ========================================
    // Model receiving borrowed funds
    // (Adapt to your flash loan provider interface)
    uint256 borrow_amount;
    require borrow_amount > 0;
    require borrow_amount <= 1000000 * 10^18;  // Realistic bound: 1M tokens
    
    // Simulate receiving borrowed funds
    // flashLoan(e_borrow, borrow_amount);
    
    // ========================================
    // STEP 2: Attack Operations
    // ========================================
    // Attacker can call any function with borrowed capital
    method f;
    require !f.isView;
    calldataarg args;
    f(e_attack, args);
    
    // ========================================
    // STEP 3: Flash Loan Repay
    // ========================================
    // Attacker must repay loan + fee
    uint256 fee = borrow_amount / 1000;  // 0.1% fee typical
    // repayFlashLoan(e_repay, borrow_amount + fee);
    
    // ========================================
    // PROFIT CHECK
    // ========================================
    mathint attacker_value_final = actor_value[attacker];
    mathint profit = attacker_value_final - attacker_value_initial;
    
    // This should FAIL if flash loan attack is profitable
    assert profit <= 0, 
        "FLASH LOAN EXPLOIT: Attacker profited from flash loan attack";
}

/**
 * @title Flash Loan Capital Amplification
 * @notice Verifies protocol is safe against amplified capital
 * @dev Key insight: Flash loans give attackers 1000x normal capital
 */
rule flash_loan_capital_amplification(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    address attacker = e.msg.sender;
    
    // Give attacker MASSIVE capital (simulates flash loan)
    require actor_value[attacker] >= 10000000 * 10^18;  // 10M tokens
    
    mathint value_before = actor_value[attacker];
    mathint protocol_before = total_system_value;
    
    calldataarg args;
    f(e, args);
    
    mathint value_after = actor_value[attacker];
    mathint protocol_after = total_system_value;
    
    mathint attacker_profit = value_after - value_before;
    mathint protocol_loss = protocol_before - protocol_after;
    
    // Even with massive capital, should not extract value
    assert attacker_profit <= 0 || protocol_loss <= 0,
        "FLASH LOAN EXPLOIT: Large capital enables value extraction";
}
```

---

## Sandwich Attack Pattern

Sandwich attacks exploit transaction ordering within a block.

```cvl
/**
 * ================================================================
 * SANDWICH ATTACK MODELING
 * ================================================================
 * 
 * Sandwich attack structure:
 *   1. Attacker sees victim's pending tx in mempool
 *   2. Attacker frontrun: Move price unfavorably for victim
 *   3. Victim tx executes at worse price
 *   4. Attacker backrun: Reverse position, capture profit
 *
 * Victim's loss = Attacker's gain (minus fees)
 * ================================================================
 */

/**
 * @title Sandwich Attack Search
 * @notice Simulates frontrun → victim → backrun sequence
 * @dev All three transactions in same block
 */
rule sandwich_attack_search(
    env e_frontrun, 
    env e_victim, 
    env e_backrun,
    method f_front,
    method f_victim,
    method f_back
) {
    // Attacker controls frontrun and backrun
    address attacker = e_frontrun.msg.sender;
    address victim = e_victim.msg.sender;
    
    require e_backrun.msg.sender == attacker;
    require attacker != victim;
    require attacker != 0;
    require victim != 0;
    
    // Same block (MEV bundle)
    require e_frontrun.block.number == e_victim.block.number;
    require e_victim.block.number == e_backrun.block.number;
    
    // Ordering: frontrun < victim < backrun
    // (Modeled implicitly by call sequence)
    
    // Filter to relevant functions (adapt to your protocol)
    require !f_front.isView && !f_victim.isView && !f_back.isView;
    
    // Capture initial state
    mathint attacker_initial = actor_value[attacker];
    mathint victim_initial = actor_value[victim];
    
    // ========================================
    // STEP 1: Frontrun
    // ========================================
    calldataarg args_front;
    f_front(e_frontrun, args_front);
    
    // ========================================
    // STEP 2: Victim Transaction
    // ========================================
    calldataarg args_victim;
    f_victim(e_victim, args_victim);
    
    // ========================================
    // STEP 3: Backrun
    // ========================================
    calldataarg args_back;
    f_back(e_backrun, args_back);
    
    // ========================================
    // PROFIT/LOSS CHECK
    // ========================================
    mathint attacker_final = actor_value[attacker];
    mathint victim_final = actor_value[victim];
    
    mathint attacker_profit = attacker_final - attacker_initial;
    mathint victim_loss = victim_initial - victim_final;
    
    // This should FAIL if sandwich is profitable
    assert attacker_profit <= 0,
        "SANDWICH EXPLOIT: Attacker profited from sandwich attack";
    
    // Additional: victim's loss should not exceed reasonable slippage
    // assert victim_loss <= victim_initial / 100,  // 1% max slippage
    //     "SANDWICH EXPLOIT: Victim suffered excessive slippage";
}

/**
 * @title Price Manipulation Check
 * @notice Verifies price cannot be manipulated then exploited
 */
rule price_manipulation_sandwich(env e1, env e2, method f1, method f2) {
    address manipulator = e1.msg.sender;
    require e2.msg.sender == manipulator;
    require e1.block.number == e2.block.number;
    
    require !f1.isView && !f2.isView;
    
    // Get price before manipulation
    // mathint price_before = getPrice();
    
    mathint value_before = actor_value[manipulator];
    
    // Step 1: Price manipulation
    calldataarg args1;
    f1(e1, args1);
    
    // mathint price_after_manipulation = getPrice();
    
    // Step 2: Exploit manipulated price
    calldataarg args2;
    f2(e2, args2);
    
    mathint value_after = actor_value[manipulator];
    
    assert value_after <= value_before,
        "PRICE MANIPULATION EXPLOIT: Manipulator profited";
}
```

---

## Staged Attack Pattern

Staged attacks accumulate position over time before triggering exploit.

```cvl
/**
 * ================================================================
 * STAGED / MULTI-BLOCK ATTACK MODELING
 * ================================================================
 * 
 * Staged attack structure:
 *   1. Accumulation phase: Build position over multiple blocks
 *   2. Trigger phase: Create conditions for exploit
 *   3. Extraction phase: Cash out accumulated advantage
 *
 * Harder to detect because each individual tx looks benign
 *
 * NOTE ON DEPTH: Real exploits (e.g., Euler $197M) can require 6+ 
 * distinct steps. This template models 3 steps as a baseline.
 * To extend depth, add more env/method/calldataarg triples and
 * chain additional phases. Each added step multiplies the search
 * space by O(N) where N = number of contract functions.
 * Use `filtered` clauses to bound to relevant functions.
 * ================================================================
 */

// Track accumulated position across phases
// Updated by rule logic (not hooks) to track attacker's growing advantage
persistent ghost mathint accumulated_position {
    init_state axiom accumulated_position == 0;
}

/**
 * @title Staged Attack Accumulation
 * @notice Models multi-block position building
 * @dev Tracks per-phase position changes to detect gradual exploitation
 */
rule staged_attack_accumulation(
    env e1, env e2, env e3,
    method f1, method f2, method f3
) {
    address attacker = e1.msg.sender;
    require e2.msg.sender == attacker;
    require e3.msg.sender == attacker;
    
    // Across multiple blocks
    require e1.block.number < e2.block.number;
    require e2.block.number < e3.block.number;
    
    require !f1.isView && !f2.isView && !f3.isView;
    
    mathint initial_value = actor_value[attacker];
    mathint initial_protocol = total_system_value;
    
    // ========================================
    // PHASE 1: First Accumulation
    // ========================================
    calldataarg args1;
    f1(e1, args1);
    
    mathint after_phase1 = actor_value[attacker];
    mathint phase1_delta = after_phase1 - initial_value;
    
    // ========================================
    // PHASE 2: Second Accumulation  
    // ========================================
    calldataarg args2;
    f2(e2, args2);
    
    mathint after_phase2 = actor_value[attacker];
    mathint phase2_delta = after_phase2 - after_phase1;
    
    // ========================================
    // PHASE 3: Extraction
    // ========================================
    calldataarg args3;
    f3(e3, args3);
    
    mathint final_value = actor_value[attacker];
    mathint final_protocol = total_system_value;
    
    mathint attacker_profit = final_value - initial_value;
    mathint protocol_loss = initial_protocol - final_protocol;
    mathint phase3_delta = final_value - after_phase2;
    
    // Multi-block attack should not be profitable
    assert attacker_profit <= 0 || protocol_loss <= 0,
        "STAGED EXPLOIT: Multi-block attack extracted value";
    
    // Additional: individual phases may look benign, but combined they aren't
    // If phase1 and phase2 are small gains but phase3 is a large extraction,
    // this is the classic "accumulate then extract" pattern
    assert !(phase1_delta >= 0 && phase2_delta >= 0 && phase3_delta > phase1_delta + phase2_delta),
        "STAGED EXPLOIT: Accumulate-then-extract pattern detected";
}

/**
 * @title Time-Delayed Attack
 * @notice Models attacks that exploit time-based vulnerabilities
 */
rule time_delayed_attack(env e_setup, env e_exploit) {
    address attacker = e_setup.msg.sender;
    require e_exploit.msg.sender == attacker;
    
    // Time must pass between setup and exploit
    require e_exploit.block.timestamp > e_setup.block.timestamp;
    require e_exploit.block.timestamp - e_setup.block.timestamp >= 86400;  // 1 day
    
    mathint initial_value = actor_value[attacker];
    
    // Setup phase
    method f_setup;
    require !f_setup.isView;
    calldataarg args_setup;
    f_setup(e_setup, args_setup);
    
    // Exploit phase (after delay)
    method f_exploit;
    require !f_exploit.isView;
    calldataarg args_exploit;
    f_exploit(e_exploit, args_exploit);
    
    mathint final_value = actor_value[attacker];
    
    assert final_value <= initial_value,
        "TIME-DELAYED EXPLOIT: Attacker profited after waiting";
}
```

---

## Governance Attack Pattern

Governance attacks exploit proposal/execution mechanisms.

```cvl
/**
 * ================================================================
 * GOVERNANCE ATTACK MODELING
 * ================================================================
 * 
 * Governance attack structure:
 *   1. Accumulate voting power (buy tokens / flash loan)
 *   2. Propose malicious action
 *   3. Vote to pass
 *   4. Wait for timelock
 *   5. Execute malicious action
 *   6. Extract value
 *
 * Defense: Sufficient timelock + voting power distribution
 * ================================================================
 */

/**
 * @title Governance Takeover Check
 * @notice Verifies governance cannot be used to extract value
 */
rule governance_extraction_check(
    env e_propose,
    env e_vote,
    env e_execute
) {
    address attacker = e_propose.msg.sender;
    
    // Attacker controls governance flow
    require e_vote.msg.sender == attacker;
    require e_execute.msg.sender == attacker;
    
    // Proper ordering with timelock
    require e_propose.block.timestamp < e_vote.block.timestamp;
    require e_vote.block.timestamp < e_execute.block.timestamp;
    
    // Timelock delay (e.g., 2 days)
    require e_execute.block.timestamp - e_vote.block.timestamp >= 172800;
    
    mathint attacker_value_before = actor_value[attacker];
    mathint protocol_value_before = total_system_value;
    
    // Simulate governance flow
    // propose(e_propose, ...);
    // vote(e_vote, ...);
    // execute(e_execute, ...);
    
    mathint attacker_value_after = actor_value[attacker];
    mathint protocol_value_after = total_system_value;
    
    // Even through governance, should not extract protocol value
    assert attacker_value_after <= attacker_value_before + protocol_value_before / 100,
        "GOVERNANCE EXPLOIT: Governance used to extract excessive value";
}
```

---

## Reentrancy Attack Pattern

```cvl
/**
 * ================================================================
 * REENTRANCY ATTACK MODELING
 * ================================================================
 */

// Track reentrancy depth
persistent ghost uint256 call_depth {
    init_state axiom call_depth == 0;
}

persistent ghost bool reentered {
    init_state axiom reentered == false;
}

/**
 * @title Reentrancy Profit Check
 * @notice Verifies reentrancy cannot be profitable
 */
rule reentrancy_no_profit(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    address attacker = e.msg.sender;
    
    // Assume attacker contract can reenter
    // (This is modeled by allowing multiple calls)
    
    mathint value_before = actor_value[attacker];
    
    calldataarg args;
    f(e, args);
    
    // If reentrancy occurred, should still not profit
    mathint value_after = actor_value[attacker];
    
    assert value_after <= value_before || !reentered,
        "REENTRANCY EXPLOIT: Reentrancy enabled profit";
}

/**
 * @title State Consistency During Callback
 * @notice Critical state should be consistent during external calls
 */
// rule state_consistent_during_callback(env e, method f) {
//     // Capture state before
//     mathint total_before = totalSupply();
//     mathint sum_before = sumOfBalances();
//     
//     // Function makes external call
//     calldataarg args;
//     f(e, args);
//     
//     // Even mid-call (during callback), invariant holds
//     // This requires modeling the callback point
// }
```

---

## Cross-Contract Attack Pattern

Real exploits often span multiple contracts (e.g., borrow from Protocol A, manipulate Protocol B, extract from Protocol A). All previous patterns operate on `currentContract` only. This section adds cross-contract attack sequences.

```cvl
/**
 * ================================================================
 * CROSS-CONTRACT ATTACK MODELING
 * ================================================================
 *
 * Cross-contract attack structure:
 *   1. Call contract A to establish position
 *   2. Call contract B to manipulate shared state / oracle
 *   3. Call contract A to extract value from manipulated state
 *
 * Requires: Both contracts linked via `using` declarations
 * Example: Protocol vault + price oracle, or lending pool + AMM
 * ================================================================
 */

// Declare linked contracts (adapt to your protocol)
// using VaultContract as vault;
// using OracleContract as oracle;
// using PoolContract as pool;

/**
 * @title Cross-Contract Attack Search
 * @notice Models attacker calling different contracts in sequence
 * @dev Requires `using` declarations for each contract
 *      Adapt method selectors to actual protocol interfaces
 */
rule cross_contract_attack_search(
    env e1, env e2, env e3
) {
    address attacker = e1.msg.sender;
    require e2.msg.sender == attacker;
    require e3.msg.sender == attacker;
    
    // Same block (atomic via flashbots bundle)
    require e1.block.number == e2.block.number;
    require e2.block.number == e3.block.number;
    
    mathint attacker_initial = actor_value[attacker];
    
    // ========================================
    // STEP 1: Interact with Contract A
    // ========================================
    // Example: vault.deposit(e1, amount);
    method f1;
    require !f1.isView;
    calldataarg args1;
    f1(e1, args1);
    
    // ========================================
    // STEP 2: Manipulate Contract B
    // ========================================
    // Example: oracle.updatePrice(e2, manipulatedPrice);
    // Example: pool.swap(e2, largeAmount);
    method f2;
    require !f2.isView;
    calldataarg args2;
    f2(e2, args2);
    
    // ========================================
    // STEP 3: Extract from Contract A
    // ========================================
    // Example: vault.withdraw(e3, inflatedAmount);
    method f3;
    require !f3.isView;
    calldataarg args3;
    f3(e3, args3);
    
    mathint attacker_final = actor_value[attacker];
    mathint profit = attacker_final - attacker_initial;
    
    assert profit <= 0,
        "CROSS-CONTRACT EXPLOIT: Attacker profited by combining contracts";
}
```

> **Adaptation Required:** Replace the generic `method f1/f2/f3` with specific contract calls:
> ```cvl
> // Instead of generic method dispatch:
> vault.deposit(e1, depositAmount);
> oracle.manipulate(e2, newPrice);
> vault.withdraw(e3, withdrawAmount);
> ```
> Generic dispatch works for single-contract rules, but cross-contract rules
> benefit from targeting specific function combinations to avoid TIMEOUT.

---

## Combinatorial Explosion Guidance

Multi-step attack rules can TIMEOUT because of $O(N^k)$ method combinations where $N$ = number of contract functions and $k$ = number of steps.

### Mitigation Strategies

| Strategy | When to Use | Example |
|----------|-------------|---------|
| **`filtered` clause** | Always for >1 step | `filtered { f1 -> f1.selector == sig:swap(...).selector }` |
| **Function pinning** | Known attack surface | Replace `method f` with `pool.swap(e, args)` |
| **`--rule` flag** | Run one rule at a time | `certoraRun config.conf --rule flash_loan_attack_search` |
| **Split by category** | Large contracts | Separate "value-moving" functions from "admin" functions |
| **Bound parameters** | Large numeric ranges | `require amount <= 10^24;` |

### Principled Depth Cutoff

| Attack Depth | Covers | Cost |
|--------------|--------|------|
| 1 step | Single-tx exploits, reentrancy | $O(N)$ — always feasible |
| 2 steps | Flash loan (borrow+exploit), setup+extract | $O(N^2)$ — feasible with filtering |
| 3 steps | Sandwich, staged accumulation | $O(N^3)$ — requires `filtered` |
| 4+ steps | Complex multi-stage (Euler-style) | Pin specific functions, don't use generic dispatch |

> **Rule of thumb:** Use parametric `method f` for depth ≤ 2. For depth ≥ 3, 
> pin at least one step to a specific function call. For depth ≥ 4, pin ALL 
> steps to specific functions based on your Phase 0 attack surface analysis.

---

## Usage Guide

### Step 1: Choose Attack Patterns Relevant to Your Protocol

| Protocol Type | Relevant Patterns |
|---------------|-------------------|
| AMM / DEX | Flash Loan, Sandwich, Price Manipulation |
| Lending | Flash Loan, Staged (interest), Liquidation |
| Staking | Staged (accumulation), Time-Delayed |
| Governance | Governance, Flash Loan (for votes) |
| NFT | Reentrancy (on receive), Staged |

### Step 2: Adapt Patterns to Your Protocol

Replace placeholder function calls with your actual protocol functions:

```cvl
// Before (template)
// flashLoan(e, amount);
// swap(e, args);

// After (your protocol)
vault.flashLoan(e, token, amount, data);
pool.swap(e, tokenIn, tokenOut, amountIn);
```

### Step 3: Run Attack Searches

```bash
# Run flash loan attack search
certoraRun config.conf --rule flash_loan_attack_search

# Run sandwich attack search
certoraRun config.conf --rule sandwich_attack_search

# Run staged attack search
certoraRun config.conf --rule staged_attack_accumulation
```

### Step 4: Interpret Results

| Result | Meaning | Action |
|--------|---------|--------|
| **VERIFIED** | No attack path found | Pattern is safe (or model incomplete) |
| **VIOLATED** | Attack path found | CE contains exploit parameters |
| **TIMEOUT** | Search space too large | Add constraints, simplify model |

---

## Integration Checklist

- [ ] Identified which attack patterns apply to my protocol
- [ ] Adapted function calls to my protocol's interface
- [ ] Set realistic bounds on attack parameters
- [ ] Applied `filtered` clauses for all multi-step rules (depth ≥ 2) ← **NEW**
- [ ] Ran flash loan pattern (if protocol holds value)
- [ ] Ran sandwich pattern (if protocol has price-sensitive operations)
- [ ] Ran staged pattern (if protocol has time-dependent state)
- [ ] Ran governance pattern (if protocol has governance)
- [ ] Ran cross-contract pattern (if protocol interacts with external contracts) ← **NEW**
- [ ] Verified no TIMEOUT; added constraints if needed ← **NEW**
- [ ] Converted any CEs to Foundry PoCs
- [ ] Documented attack surfaces in security report
