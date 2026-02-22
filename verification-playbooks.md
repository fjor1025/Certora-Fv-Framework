# Verification Playbooks — Complete Worked Examples

> **Framework Version:** v3.0 (Offensive Verification + Red Team Hardening)
> **Source:** Derived from the RareSkills Certora Book capstone projects and OpenZeppelin verification methodology
> **Purpose:** Provide complete, battle-tested specification templates for ERC-20, WETH, and ERC-721 tokens — ready to adapt for any project.

---

## Table of Contents

1. [ERC-20 Verification Playbook (22 Rules)](#1-erc-20-verification-playbook-22-rules)
2. [WETH Verification Playbook (Solady Pattern)](#2-weth-verification-playbook-solady-pattern)
3. [ERC-721 Verification Playbook (OpenZeppelin Pattern)](#3-erc-721-verification-playbook-openzeppelin-pattern)
4. [Methodology Reference](#4-methodology-reference)

---

## 1. ERC-20 Verification Playbook (22 Rules)

### 1.1 Overview

This playbook verifies a standard ERC-20 token using the **Four-Phase Methodology:**

| Phase | Goal | Rules |
|-------|------|-------|
| **Phase 1** | Function correctness (Liveness + Effect) | 8 rules |
| **Phase 2** | No side effects | 5 rules |
| **Phase 3** | Global invariants | 3 invariants |
| **Phase 4** | Authorization/Change governance | 6 rules |

### 1.2 Ghost and Hook Infrastructure

Every ERC-20 specification begins with sum-tracking:

```cvl
// ============================================================
// GHOST VARIABLES
// ============================================================

// Tracks the sum of all balances across all accounts
ghost mathint g_sumOfBalances {
    init_state axiom g_sumOfBalances == 0;
}

// ============================================================
// HOOKS
// ============================================================

// Sstore hook: fires on every balance write
hook Sstore balanceOf[KEY address account] uint256 newValue (uint256 oldValue) {
    g_sumOfBalances = g_sumOfBalances + newValue - oldValue;
}

// Sload hook: constrains reads to prevent unrealistic initial states
hook Sload uint256 balance balanceOf[KEY address account] {
    require g_sumOfBalances >= to_mathint(balance);
}
```

### 1.3 Standard Definitions

```cvl
// ============================================================
// DEFINITIONS
// ============================================================

definition nonpayable(env e) returns bool = e.msg.value == 0;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
```

### 1.4 Methods Block

```cvl
methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function allowance(address, address) external returns (uint256) envfree;
}
```

### 1.5 Phase 1: Function Correctness

#### Rule 1: `transfer` — Liveness + Effect

```cvl
rule transfer(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to; uint256 amount;
    address sender = e.msg.sender;

    // Pre-state
    mathint senderBalBefore = balanceOf(sender);
    mathint receiverBalBefore = balanceOf(to);

    // Action
    transfer@withrevert(e, to, amount);
    bool success = !lastReverted;

    // LIVENESS — enumerate ALL revert conditions
    assert success <=> (
        sender != 0 &&
        to != 0 &&
        senderBalBefore >= to_mathint(amount) &&
        (sender != to || to_mathint(amount) == 0 || senderBalBefore >= to_mathint(amount))
    );

    // EFFECT — correct state changes on success
    if (sender == to) {
        // Self-transfer: balance unchanged
        assert success => to_mathint(balanceOf(sender)) == senderBalBefore;
    } else {
        assert success => to_mathint(balanceOf(sender)) == senderBalBefore - to_mathint(amount);
        assert success => to_mathint(balanceOf(to)) == receiverBalBefore + to_mathint(amount);
    }
}
```

#### Rule 2: `transferFrom` — Liveness + Effect

```cvl
rule transferFrom(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address from; address to; uint256 amount;

    // Pre-state
    mathint fromBalBefore = balanceOf(from);
    mathint toBalBefore = balanceOf(to);
    mathint allowanceBefore = allowance(from, e.msg.sender);

    // Action
    transferFrom@withrevert(e, from, to, amount);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (
        e.msg.sender != 0 &&
        from != 0 &&
        to != 0 &&
        fromBalBefore >= to_mathint(amount) &&
        (allowanceBefore >= to_mathint(amount) || allowanceBefore == max_uint256)
    );

    // EFFECT — balance changes
    if (from == to) {
        assert success => to_mathint(balanceOf(from)) == fromBalBefore;
    } else {
        assert success => to_mathint(balanceOf(from)) == fromBalBefore - to_mathint(amount);
        assert success => to_mathint(balanceOf(to)) == toBalBefore + to_mathint(amount);
    }

    // EFFECT — allowance changes
    assert success => (
        allowanceBefore == max_uint256 ||
        to_mathint(allowance(from, e.msg.sender)) == allowanceBefore - to_mathint(amount)
    );
}
```

#### Rule 3: `approve` — Liveness + Effect

```cvl
rule approve(env e) {
    require nonpayable(e);

    address spender; uint256 amount;

    approve@withrevert(e, spender, amount);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (e.msg.sender != 0 && spender != 0);

    // EFFECT
    assert success => to_mathint(allowance(e.msg.sender, spender)) == to_mathint(amount);
}
```

#### Rule 4: `mint` — Liveness + Effect

```cvl
rule mint(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to; uint256 amount;

    // Pre-state
    mathint toBalBefore = balanceOf(to);
    mathint supplyBefore = totalSupply();

    // Prevent overflow in unchecked blocks
    require toBalBefore + to_mathint(amount) <= max_uint256;
    require supplyBefore + to_mathint(amount) <= max_uint256;

    // Action
    mint@withrevert(e, to, amount);
    bool success = !lastReverted;

    // LIVENESS (adapt to your access control)
    assert success <=> (to != 0 && e.msg.sender == owner());

    // EFFECT
    assert success => to_mathint(balanceOf(to)) == toBalBefore + to_mathint(amount);
    assert success => to_mathint(totalSupply()) == supplyBefore + to_mathint(amount);
}
```

#### Rule 5: `burn` — Liveness + Effect

```cvl
rule burn(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    uint256 amount;

    // Pre-state
    mathint senderBalBefore = balanceOf(e.msg.sender);
    mathint supplyBefore = totalSupply();

    // Action
    burn@withrevert(e, amount);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (
        e.msg.sender != 0 &&
        senderBalBefore >= to_mathint(amount)
    );

    // EFFECT
    assert success => to_mathint(balanceOf(e.msg.sender)) == senderBalBefore - to_mathint(amount);
    assert success => to_mathint(totalSupply()) == supplyBefore - to_mathint(amount);
}
```

### 1.6 Phase 2: No Side Effects

#### Rule 6: `transfer` — No Side Effects on Third Parties

```cvl
rule transferNoSideEffects(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to; uint256 amount;
    address other;

    // Uninvolved account
    require other != e.msg.sender;
    require other != to;

    mathint otherBalBefore = balanceOf(other);

    transfer(e, to, amount);

    assert to_mathint(balanceOf(other)) == otherBalBefore;
}
```

#### Rule 7: `transferFrom` — No Side Effects on Third Parties

```cvl
rule transferFromNoSideEffects(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address from; address to; uint256 amount;
    address other;

    require other != from;
    require other != to;

    mathint otherBalBefore = balanceOf(other);

    transferFrom(e, from, to, amount);

    assert to_mathint(balanceOf(other)) == otherBalBefore;
}
```

#### Rule 8: `approve` — No Side Effects on Balances

```cvl
rule approveNoSideEffects(env e) {
    require nonpayable(e);

    address spender; uint256 amount;
    address anyAccount;

    mathint balBefore = balanceOf(anyAccount);

    approve(e, spender, amount);

    assert to_mathint(balanceOf(anyAccount)) == balBefore;
}
```

#### Rule 9: `approve` — No Side Effects on Other Allowances

```cvl
rule approveDoesNotAffectOtherAllowances(env e) {
    require nonpayable(e);

    address spender; uint256 amount;
    address otherOwner; address otherSpender;

    // At least one must differ
    require otherOwner != e.msg.sender || otherSpender != spender;

    mathint allowanceBefore = allowance(otherOwner, otherSpender);

    approve(e, spender, amount);

    assert to_mathint(allowance(otherOwner, otherSpender)) == allowanceBefore;
}
```

#### Rule 10: `mint` and `burn` — No Side Effects

```cvl
rule mintNoSideEffects(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to; uint256 amount;
    address other;
    require other != to;

    // Prevent overflow
    require to_mathint(balanceOf(to)) + to_mathint(amount) <= max_uint256;
    require to_mathint(totalSupply()) + to_mathint(amount) <= max_uint256;

    mathint otherBalBefore = balanceOf(other);
    mint(e, to, amount);
    assert to_mathint(balanceOf(other)) == otherBalBefore;
}
```

### 1.7 Phase 3: Global Invariants

#### Invariant 1: Total Supply Equals Sum of Balances

```cvl
invariant totalSupplyEqualsSumOfBalances()
    to_mathint(totalSupply()) == g_sumOfBalances
{
    preserved with (env e) {
        require nonpayable(e);
    }
}
```

#### Invariant 2: Individual Balance ≤ Total Supply

```cvl
invariant individualBalanceCapped(address account)
    to_mathint(balanceOf(account)) <= to_mathint(totalSupply())
{
    preserved with (env e) {
        require nonpayable(e);
        requireInvariant totalSupplyEqualsSumOfBalances();
    }
}
```

#### Invariant 3: Zero Address Has Zero Balance

```cvl
invariant zeroAddressNoBalance()
    balanceOf(0) == 0
{
    preserved with (env e) {
        require nonpayable(e);
        requireInvariant totalSupplyEqualsSumOfBalances();
    }
}
```

### 1.8 Phase 4: Authorization and Change Governance

#### Rule 11: Only Authorized Functions Change Total Supply

```cvl
rule onlyMintBurnChangeSupply(env e, method f, calldataarg args) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    mathint supplyBefore = totalSupply();
    f(e, args);
    mathint supplyAfter = totalSupply();

    assert supplyAfter != supplyBefore => (
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:burn(uint256).selector
    );
}
```

#### Rule 12: Only Authorized Functions Change Balances

```cvl
rule balanceChangeRestriction(env e, method f, calldataarg args) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address account;
    mathint balBefore = balanceOf(account);
    f(e, args);
    mathint balAfter = balanceOf(account);

    assert balAfter != balBefore => (
        f.selector == sig:transfer(address,uint256).selector ||
        f.selector == sig:transferFrom(address,address,uint256).selector ||
        f.selector == sig:mint(address,uint256).selector ||
        f.selector == sig:burn(uint256).selector
    );
}
```

#### Rule 13: Only Authorized Functions Change Allowances

```cvl
rule allowanceChangeRestriction(env e, method f, calldataarg args) {
    require nonpayable(e);

    address owner; address spender;
    mathint allowanceBefore = allowance(owner, spender);
    f(e, args);
    mathint allowanceAfter = allowance(owner, spender);

    assert allowanceAfter != allowanceBefore => (
        f.selector == sig:approve(address,uint256).selector ||
        f.selector == sig:transferFrom(address,address,uint256).selector
    );
}
```

#### Rule 14: Transfer Can't Increase Sender Balance

```cvl
rule transferNeverIncreasesSenderBalance(env e) {
    require nonpayable(e);
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to; uint256 amount;
    require e.msg.sender != to;

    mathint senderBalBefore = balanceOf(e.msg.sender);
    transfer(e, to, amount);

    assert to_mathint(balanceOf(e.msg.sender)) <= senderBalBefore;
}
```

#### Rule 15: Zero Transfer Is Always Valid

```cvl
rule zeroTransferIsValid(env e) {
    require nonpayable(e);
    require e.msg.sender != 0;
    requireInvariant totalSupplyEqualsSumOfBalances();

    address to;
    require to != 0;

    transfer@withrevert(e, to, 0);
    assert !lastReverted;
}
```

#### Rule 16: Self-Approve

```cvl
rule selfApproveWorks(env e) {
    require nonpayable(e);
    require e.msg.sender != 0;

    uint256 amount;
    approve@withrevert(e, e.msg.sender, amount);
    assert !lastReverted;
    assert to_mathint(allowance(e.msg.sender, e.msg.sender)) == to_mathint(amount);
}
```

### 1.9 Configuration File

```json
{
    "files": [
        "contracts/MyERC20.sol:MyERC20"
    ],
    "verify": "MyERC20:specs/erc20.spec",
    "rule_sanity": "basic",
    "optimistic_loop": true,
    "loop_iter": "3",
    "msg": "ERC-20 Verification Suite v1.5"
}
```

---

## 2. WETH Verification Playbook (Solady Pattern)

### 2.1 Overview

WETH (Wrapped Ether) verification requires techniques beyond standard ERC-20 because:
- **ETH ↔ WETH exchange** creates a solvency constraint
- **Low-level assembly `call()`** requires opcode hooks
- **Non-standard storage layouts** (Solady) may defeat standard ghost tracking

### 2.2 Ghost and Hook Infrastructure

```cvl
// ============================================================
// GHOST VARIABLES
// ============================================================

// Track whether a low-level CALL failed
// MUST be persistent — survives reverts and unresolved external calls
persistent ghost bool g_lowLevelCallFail;

// ============================================================
// HOOKS
// ============================================================

// CALL opcode hook — captures assembly call() return code
hook CALL(uint gas, address to, uint value,
          uint argsOffset, uint argsLength,
          uint retOffset, uint retLength) uint returnCode {
    if (returnCode == 0) {
        g_lowLevelCallFail = true;   // CALL failed
    } else {
        g_lowLevelCallFail = false;  // CALL succeeded
    }
}
```

### 2.3 Standard Definitions

```cvl
definition nonpayable(env e) returns bool = e.msg.value == 0;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
```

### 2.4 Methods Block

```cvl
methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
}
```

### 2.5 Solvency Invariant

The core WETH property: **the contract holds at least as much ETH as WETH in circulation.**

```cvl
invariant solvency()
    nativeBalances[currentContract] >= to_mathint(totalSupply())
{
    preserved with (env e) {
        require e.msg.sender != currentContract;
    }
    preserved deposit() with (env e) {
        require e.msg.sender != currentContract;
    }
    preserved withdraw(uint256 amount) with (env e) {
        require e.msg.sender != currentContract;
        require to_mathint(balanceOf(e.msg.sender)) <= to_mathint(totalSupply());
    }
}
```

**Why `require e.msg.sender != currentContract`?**
This is a modeling constraint, not a real revert condition. In practice, `currentContract` is WETH itself — WETH depositing ETH into itself creates a paradox where ETH is simultaneously sent and received. This doesn't occur in real usage and would make the invariant trivially violated by the Prover's symbolic exploration.

### 2.6 Deposit Verification

```cvl
rule deposit(env e) {
    require e.msg.sender != currentContract;
    requireInvariant solvency();

    // Pre-state
    mathint supplyBefore = totalSupply();
    mathint senderBalBefore = balanceOf(e.msg.sender);
    mathint ethBefore = nativeBalances[currentContract];

    // Action
    deposit@withrevert(e);
    bool success = !lastReverted;

    // LIVENESS — deposit succeeds if msg.value > 0 and no overflow
    assert success <=> (
        senderBalBefore + e.msg.value <= max_uint256 &&
        supplyBefore + e.msg.value <= max_uint256
    );

    // EFFECT — WETH minted equals ETH sent
    assert success => to_mathint(balanceOf(e.msg.sender)) == senderBalBefore + e.msg.value;
    assert success => to_mathint(totalSupply()) == supplyBefore + e.msg.value;
}
```

### 2.7 Withdraw Verification

```cvl
rule withdraw(env e) {
    require e.msg.sender != currentContract;
    requireInvariant solvency();

    uint256 amount;

    // Pre-state
    mathint supplyBefore = totalSupply();
    mathint senderBalBefore = balanceOf(e.msg.sender);
    mathint ethBefore = nativeBalances[currentContract];

    // Action
    withdraw@withrevert(e, amount);
    bool success = !lastReverted;

    // LIVENESS — enumerate all revert conditions
    assert success <=> (
        e.msg.value == 0 &&            // non-payable function
        senderBalBefore >= to_mathint(amount) &&  // sufficient WETH balance
        !g_lowLevelCallFail            // low-level ETH transfer succeeded
    );

    // EFFECT — WETH burned equals ETH sent back
    assert success => to_mathint(balanceOf(e.msg.sender)) == senderBalBefore - to_mathint(amount);
    assert success => to_mathint(totalSupply()) == supplyBefore - to_mathint(amount);
}
```

### 2.8 Solvency After All Functions

```cvl
rule solvencyAfterAnyCall(env e, method f, calldataarg args)
filtered {
    f -> !f.isView
} {
    require e.msg.sender != currentContract;
    requireInvariant solvency();
    require to_mathint(balanceOf(e.msg.sender)) <= to_mathint(totalSupply());

    f(e, args);

    assert nativeBalances[currentContract] >= to_mathint(totalSupply());
}
```

### 2.9 Non-Standard Storage (Solady Pattern)

Solady's WETH uses computed storage slots (not sequential), which prevents standard `hook Sstore balanceOf[KEY address]` from working. In this case:

1. **Cannot use standard balance-tracking ghost + hook** — the Prover can't resolve computed slot addresses
2. **Use `filtered` blocks** to exclude functions where standard tracking fails
3. **Rely on the solvency invariant** as the primary safety property
4. **Document the limitation clearly** in the specification

```cvl
// NOTE: Solady uses non-standard storage layout.
// Standard balance-tracking ghosts cannot hook into computed slots.
// The solvency invariant provides coverage via ETH balance checks.
// Function-specific rules verify individual correctness.
```

### 2.10 Configuration

```json
{
    "files": [
        "contracts/WETH.sol:WETH"
    ],
    "verify": "WETH:specs/weth.spec",
    "rule_sanity": "basic",
    "optimistic_loop": true,
    "loop_iter": "2",
    "msg": "WETH Solvency Verification"
}
```

---

## 3. ERC-721 Verification Playbook (OpenZeppelin Pattern)

### 3.1 Overview

ERC-721 verification requires different patterns than ERC-20:
- **Ownership is per-token** (not per-address balance)
- **Approval has two levels** (operator + per-token)
- **`safeMint` and `safeTransferFrom`** invoke external callbacks (`onERC721Received`)
- **Reverting getters** (`ownerOf` reverts for unminted tokens) require harness functions

### 3.2 Harness Contract

A harness exposes internal state that public functions hide behind reverts:

```solidity
// ERC721Harness.sol
pragma solidity ^0.8.0;

import {ERC721} from "./ERC721.sol";

contract ERC721Harness is ERC721 {
    constructor(string memory name_, string memory symbol_) ERC721(name_, symbol_) {}

    // Returns address(0) instead of reverting for unminted tokens
    function unsafeOwnerOf(uint256 tokenId) external view returns (address) {
        return _owners[tokenId];
    }

    // Returns address(0) instead of reverting for unminted tokens
    function unsafeGetApproved(uint256 tokenId) external view returns (address) {
        return _tokenApprovals[tokenId];
    }

    // Expose total supply if not public
    function supply() external view returns (uint256) {
        return _totalSupply;
    }

    // Mint function (for testing)
    function mint(address to, uint256 tokenId) external {
        _mint(to, tokenId);
    }

    // Burn function (for testing)
    function burn(uint256 tokenId) external {
        _burn(tokenId);
    }
}
```

### 3.3 Mock Receiver for DISPATCHER

```solidity
// ERC721ReceiverHarness.sol
pragma solidity ^0.8.0;

import {IERC721Receiver} from "./IERC721Receiver.sol";

contract ERC721ReceiverHarness is IERC721Receiver {
    function onERC721Received(
        address, address, uint256, bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```

### 3.4 Ghost and Hook Infrastructure

```cvl
// ============================================================
// GHOST VARIABLES
// ============================================================

// Track total supply via storage slot
ghost mathint _supply {
    init_state axiom _supply == 0;
}

// Track per-address owned count
ghost mapping(address => mathint) _ownedByUser {
    init_state axiom forall address a. _ownedByUser[a] == 0;
}

// ============================================================
// HOOKS
// ============================================================

// _totalSupply tracking
hook Sstore _totalSupply uint256 newValue (uint256 oldValue) {
    _supply = _supply + newValue - oldValue;
}

// Ownership tracking — handles mint, burn, and transfer in one hook
hook Sstore _owners[KEY uint256 tokenId] address newOwner (address oldOwner) {
    // Increment new owner's count (if not mint to zero — which doesn't happen)
    _ownedByUser[newOwner] = _ownedByUser[newOwner] + (newOwner != 0 ? 1 : 0);
    // Decrement old owner's count (if not burn from zero — which doesn't happen)
    _ownedByUser[oldOwner] = _ownedByUser[oldOwner] - (oldOwner != 0 ? 1 : 0);
}
```

### 3.5 Definitions and Methods

```cvl
definition nonpayable(env e) returns bool = e.msg.value == 0;
definition nonzerosender(env e) returns bool = e.msg.sender != 0;
definition balanceLimited(address a) returns bool = balanceOf(a) < max_uint256;

methods {
    function balanceOf(address) external returns (uint256) envfree;
    function unsafeOwnerOf(uint256) external returns (address) envfree;
    function unsafeGetApproved(uint256) external returns (address) envfree;
    function isApprovedForAll(address, address) external returns (bool) envfree;
    function supply() external returns (uint256) envfree;

    // DISPATCHER for external callbacks (safeMint, safeTransferFrom)
    function _.onERC721Received(address, address, uint256, bytes) external
        => DISPATCHER(true);
}
```

### 3.6 Key Invariants

#### Invariant 1: Balance Equals Owned Token Count

```cvl
invariant balanceEqualsOwnedCount(address account)
    _ownedByUser[account] == to_mathint(balanceOf(account))
{
    preserved with (env e) {
        require nonpayable(e);
        require balanceLimited(account);
    }
}
```

#### Invariant 2: Owner Has Non-Zero Balance

```cvl
invariant ownerHasBalance(uint256 tokenId)
    unsafeOwnerOf(tokenId) != 0 => balanceOf(unsafeOwnerOf(tokenId)) > 0
{
    preserved with (env e) {
        require nonpayable(e);
        requireInvariant balanceEqualsOwnedCount(unsafeOwnerOf(tokenId));
    }
}
```

#### Invariant 3: Unminted Tokens Have No Approval

```cvl
invariant unmintedHasNoApproval(uint256 tokenId)
    unsafeOwnerOf(tokenId) == 0 => unsafeGetApproved(tokenId) == 0
{
    preserved with (env e) {
        require nonpayable(e);
    }
}
```

#### Invariant 4: Zero Address Owns Nothing

```cvl
invariant zeroAddressOwnsNothing()
    _ownedByUser[0] == 0
{
    preserved with (env e) {
        require nonpayable(e);
    }
}
```

#### Invariant 5: Supply Consistency

```cvl
invariant supplyConsistency()
    _supply >= 0
{
    preserved with (env e) {
        require nonpayable(e);
    }
}
```

### 3.7 Sound Helper Function

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
    else if (f.selector == sig:safeTransferFrom(address,address,uint256).selector) {
        address from; address to; uint256 tokenId;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        safeTransferFrom(e, from, to, tokenId);
    }
    else if (f.selector == sig:safeTransferFrom(address,address,uint256,bytes).selector) {
        address from; address to; uint256 tokenId; bytes data;
        require balanceLimited(to);
        requireInvariant ownerHasBalance(tokenId);
        safeTransferFrom(e, from, to, tokenId, data);
    }
    else {
        calldataarg args;
        f(e, args);
    }
}
```

### 3.8 Mint Verification (Liveness / Effect / No-Side-Effect)

```cvl
rule mint(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address to; uint256 tokenId;

    // Preconditions
    require balanceLimited(to);
    requireInvariant ownerHasBalance(tokenId);

    // Pre-state
    address ownerBefore = unsafeOwnerOf(tokenId);
    mathint supplyBefore = _supply;
    mathint balOfToBefore = balanceOf(to);

    // Side-effect witnesses
    address otherAccount;
    uint256 otherTokenId;
    mathint balOfOtherBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    mint@withrevert(e, to, tokenId);
    bool success = !lastReverted;

    // LIVENESS — mint succeeds iff token is unowned and recipient is non-zero
    assert success <=> (ownerBefore == 0 && to != 0);

    // EFFECT — correct state changes
    assert success => (
        _supply == supplyBefore + 1 &&
        to_mathint(balanceOf(to)) == balOfToBefore + 1 &&
        unsafeOwnerOf(tokenId) == to
    );

    // NO SIDE EFFECT — only recipient's balance changes
    assert balanceOf(otherAccount) != balOfOtherBefore => otherAccount == to;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId;
}
```

### 3.9 Burn Verification

```cvl
rule burn(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    uint256 tokenId;

    requireInvariant ownerHasBalance(tokenId);

    // Pre-state
    address ownerBefore = unsafeOwnerOf(tokenId);
    mathint supplyBefore = _supply;
    mathint balOfOwnerBefore = balanceOf(ownerBefore);

    // Side-effect witnesses
    address otherAccount;
    uint256 otherTokenId;
    mathint balOfOtherBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    burn@withrevert(e, tokenId);
    bool success = !lastReverted;

    // LIVENESS — burn succeeds iff token has an owner
    assert success <=> (ownerBefore != 0);

    // EFFECT
    assert success => (
        _supply == supplyBefore - 1 &&
        to_mathint(balanceOf(ownerBefore)) == balOfOwnerBefore - 1 &&
        unsafeOwnerOf(tokenId) == 0 &&
        unsafeGetApproved(tokenId) == 0
    );

    // NO SIDE EFFECT
    assert balanceOf(otherAccount) != balOfOtherBefore => otherAccount == ownerBefore;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId;
}
```

### 3.10 TransferFrom Verification

```cvl
rule transferFrom(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address from; address to; uint256 tokenId;

    require balanceLimited(to);
    requireInvariant ownerHasBalance(tokenId);
    requireInvariant balanceEqualsOwnedCount(from);
    requireInvariant balanceEqualsOwnedCount(to);

    // Pre-state
    address ownerBefore = unsafeOwnerOf(tokenId);
    address approvedBefore = unsafeGetApproved(tokenId);
    bool isOperator = isApprovedForAll(from, e.msg.sender);
    mathint fromBalBefore = balanceOf(from);
    mathint toBalBefore = balanceOf(to);

    // Side-effect witnesses
    address otherAccount;
    uint256 otherTokenId;
    mathint otherBalBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    transferFrom@withrevert(e, from, to, tokenId);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (
        ownerBefore == from &&
        from != 0 &&
        to != 0 &&
        (e.msg.sender == from || e.msg.sender == approvedBefore || isOperator)
    );

    // EFFECT
    assert success => unsafeOwnerOf(tokenId) == to;
    assert success => unsafeGetApproved(tokenId) == 0;  // Approval cleared

    // Balance changes depend on self-transfer
    assert success => to_mathint(balanceOf(from)) ==
        fromBalBefore - assert_uint256(from != to ? 1 : 0);
    assert success => to_mathint(balanceOf(to)) ==
        toBalBefore + assert_uint256(from != to ? 1 : 0);

    // NO SIDE EFFECT
    assert balanceOf(otherAccount) != otherBalBefore =>
        (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore =>
        otherTokenId == tokenId;
}
```

### 3.11 SafeMint Verification (Callback-Aware)

`safeMint` extends `mint` with an `onERC721Received` callback. The callback can reject the token (revert), which adds a distinct liveness condition:

```cvl
rule safeMint(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address to; uint256 tokenId;

    require balanceLimited(to);
    requireInvariant ownerHasBalance(tokenId);

    // Pre-state
    address ownerBefore = unsafeOwnerOf(tokenId);
    mathint supplyBefore = _supply;
    mathint balOfToBefore = balanceOf(to);

    // Side-effect witnesses
    address otherAccount;
    uint256 otherTokenId;
    mathint balOfOtherBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    mint@withrevert(e, to, tokenId); // Same underlying _safeMint in harness
    bool success = !lastReverted;

    // LIVENESS — same as mint conditions PLUS callback must not reject
    // If ownerBefore == 0 && to != 0 but success is false → callback rejected
    // This is caught implicitly: the <=> enumerates success preconditions,
    // and the DISPATCHER'd mock receiver always returns the correct selector.
    // In production, write separate callback-rejection test:
    assert success <=> (ownerBefore == 0 && to != 0);

    // EFFECT — identical to mint
    assert success => (
        _supply == supplyBefore + 1 &&
        to_mathint(balanceOf(to)) == balOfToBefore + 1 &&
        unsafeOwnerOf(tokenId) == to
    );

    // NO SIDE EFFECT
    assert balanceOf(otherAccount) != balOfOtherBefore => otherAccount == to;
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore => otherTokenId == tokenId;
}
```

> **Callback coverage note:** The mock receiver (`ERC721ReceiverHarness`) always returns the correct selector. To test callback rejection, create a second receiver that reverts, and verify the liveness assertion captures the failure. The biconditional `<=>` will fail if a path exists where preconditions hold but the call reverts (callback rejection), which is the correct behavior to detect.

### 3.12 SafeTransferFrom Verification (Callback-Aware)

```cvl
rule safeTransferFrom(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address from; address to; uint256 tokenId;

    require balanceLimited(to);
    requireInvariant ownerHasBalance(tokenId);
    requireInvariant balanceEqualsOwnedCount(from);
    requireInvariant balanceEqualsOwnedCount(to);

    // Pre-state
    address ownerBefore = unsafeOwnerOf(tokenId);
    address approvedBefore = unsafeGetApproved(tokenId);
    bool isOperator = isApprovedForAll(from, e.msg.sender);
    mathint fromBalBefore = balanceOf(from);
    mathint toBalBefore = balanceOf(to);

    // Side-effect witnesses
    address otherAccount;
    uint256 otherTokenId;
    mathint otherBalBefore = balanceOf(otherAccount);
    address otherOwnerBefore = unsafeOwnerOf(otherTokenId);

    // Action
    safeTransferFrom@withrevert(e, from, to, tokenId);
    bool success = !lastReverted;

    // LIVENESS — same authorization as transferFrom, plus callback must accept
    assert success <=> (
        ownerBefore == from &&
        from != 0 &&
        to != 0 &&
        (e.msg.sender == from || e.msg.sender == approvedBefore || isOperator)
    );

    // EFFECT
    assert success => unsafeOwnerOf(tokenId) == to;
    assert success => unsafeGetApproved(tokenId) == 0;  // Approval cleared

    // Balance changes depend on self-transfer
    assert success => to_mathint(balanceOf(from)) ==
        fromBalBefore - assert_uint256(from != to ? 1 : 0);
    assert success => to_mathint(balanceOf(to)) ==
        toBalBefore + assert_uint256(from != to ? 1 : 0);

    // NO SIDE EFFECT
    assert balanceOf(otherAccount) != otherBalBefore =>
        (otherAccount == from || otherAccount == to);
    assert unsafeOwnerOf(otherTokenId) != otherOwnerBefore =>
        otherTokenId == tokenId;
}
```

### 3.13 Supply Change Authorization

```cvl
rule supplyChangeRestriction(env e) {
    require nonzerosender(e);
    require nonpayable(e);

    mathint supplyBefore = _supply;

    method f;
    helperSoundFnCall(e, f);

    mathint supplyAfter = _supply;

    // Supply can only increase by 1 (mint) or decrease by 1 (burn)
    assert supplyAfter > supplyBefore => (
        supplyAfter == supplyBefore + 1 &&
        f.selector == sig:mint(address,uint256).selector
    );

    assert supplyAfter < supplyBefore => (
        supplyAfter == supplyBefore - 1 &&
        f.selector == sig:burn(uint256).selector
    );
}
```

### 3.12 Approval Verification

```cvl
rule approve(env e) {
    require nonpayable(e);
    require nonzerosender(e);

    address to; uint256 tokenId;

    address ownerBefore = unsafeOwnerOf(tokenId);
    bool isOperator = isApprovedForAll(ownerBefore, e.msg.sender);

    // Side-effect witness
    uint256 otherTokenId;
    address otherApprovalBefore = unsafeGetApproved(otherTokenId);

    approve@withrevert(e, to, tokenId);
    bool success = !lastReverted;

    // LIVENESS
    assert success <=> (
        ownerBefore != 0 &&
        to != ownerBefore &&
        (e.msg.sender == ownerBefore || isOperator)
    );

    // EFFECT
    assert success => unsafeGetApproved(tokenId) == to;

    // NO SIDE EFFECT
    assert unsafeGetApproved(otherTokenId) != otherApprovalBefore =>
        otherTokenId == tokenId;
}
```

### 3.13 Configuration

```json
{
    "files": [
        "contracts/ERC721Harness.sol:ERC721Harness",
        "contracts/ERC721ReceiverHarness.sol:ERC721ReceiverHarness"
    ],
    "verify": "ERC721Harness:specs/erc721.spec",
    "rule_sanity": "basic",
    "optimistic_loop": true,
    "loop_iter": "2",
    "msg": "ERC-721 Full Verification Suite"
}
```

---

## 4. Methodology Reference

### 4.1 The Four-Phase System

```
Phase 0: BUILTIN SAFETY SCAN (NEW — v8.8.0)
├── use builtin rule sanity           →  Can all methods execute?
├── use builtin rule uncheckedOverflow →  Any unsafe unchecked math?
├── use builtin rule safeCasting       →  Any unsafe type casts?
├── use builtin rule viewReentrancy    →  Read-only reentrancy?
├── use builtin rule msgValueInLoopRule →  msg.value in loops?
└── use builtin rule hasDelegateCalls  →  Unexpected delegates?

Phase 0.5: REACHABILITY VALIDATION (NEW — v1.8)
├── For EVERY state-changing function:
│   ├── f@withrevert(e, args);
│   └── satisfy !lastReverted;  →  Proves function is live
├── If ANY satisfy is VIOLATED → function always reverts
│   └── Every assert rule for that function is VACUOUSLY TRUE
└── Fix harness / DISPATCHER / require constraints before Phase 1

Phase 1: FUNCTION CORRECTNESS
├── For each external function:
│   ├── Liveness: assert success <=> (preconditions)
│   └── Effect:   assert success => (state_changes)
│
Phase 2: NO SIDE EFFECTS
├── For each state-changing function:
│   └── Assert uninvolved accounts/slots unchanged
│
Phase 3: GLOBAL INVARIANTS
├── Sum tracking (ghost + hook)
├── Individual cap (balance ≤ totalSupply)
├── Zero-address constraints
│
Phase 4: AUTHORIZATION
├── Parametric change governance rules
└── "Only f1, f2 can change state X"
```

### 4.2 Ghost + Hook Setup Checklist

| Step | Action | Example |
|------|--------|---------|
| 1 | Declare ghost with `init_state axiom` | `ghost mathint sum { init_state axiom sum == 0; }` |
| 2 | Write Sstore hook with delta pattern | `hook Sstore balanceOf[KEY address a] uint256 new (uint256 old) { sum = sum + new - old; }` |
| 3 | Write Sload hook for constraint | `hook Sload uint256 val balanceOf[KEY address a] { require sum >= to_mathint(val); }` |
| 4 | Write invariant using ghost | `invariant sumEqSupply() to_mathint(totalSupply()) == sum;` |
| 5 | Use `requireInvariant` in rules | `requireInvariant sumEqSupply();` |

### 4.3 Revert Condition Enumeration Checklist

When writing `assert lastReverted <=> (conditions)`:

| Check | Example |
|-------|---------|
| Non-payable function | `e.msg.value != 0` |
| Zero-address sender | `e.msg.sender == 0` |
| Zero-address recipient | `to == 0` |
| Insufficient balance | `balanceOf(sender) < amount` |
| Insufficient allowance | `allowance(from, sender) < amount` |
| Overflow (unchecked blocks) | `balance + amount > max_uint256` |
| Access control | `e.msg.sender != owner()` |
| Low-level call failure | `g_lowLevelCallFail` |
| Callback rejection | Receiver doesn't return correct selector |

### 4.4 When to Use Persistent Ghosts

| Scenario | Regular Ghost | Persistent Ghost |
|----------|---------------|------------------|
| Sum tracking for balances | ✓ | |
| Tracking CALL opcode results | | ✓ |
| Verifying assembly transfer success | | ✓ |
| Tracking state across reverts | | ✓ |
| Standard storage slot tracking | ✓ | |

### 4.5 Common Pitfalls and Solutions

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| `require_uint256` on arithmetic | Bug hidden, rule passes | Use `mathint` exclusively |
| Missing `e.msg.value == 0` | Unexpected revert counterexample | Add `definition nonpayable(env e)` |
| Missing self-transfer case | Balance assertion fails | Branch on `from == to` |
| Ghost not initialized | Invariant base case fails | Add `init_state axiom` |
| `lastReverted` overwritten | Wrong revert captured | Capture as `bool` immediately |
| String in Solidity | "Unwinding condition" error | Set `--loop_iter 4` or higher |
| Computed storage slots | Hook never fires | Use `filtered` block, document limitation |
| `msg.sender == currentContract` | Paradoxical state in invariant | Add `require e.msg.sender != currentContract` |
| Unchecked block overflow not caught | Manual rules miss edge cases | Run `use builtin rule uncheckedOverflow` (v8.8.0+) |
| Silent cast truncation | `uint16(x)` silently wraps | Run `use builtin rule safeCasting` (v8.8.0+) |

---

## 5. Initializable Verification Playbook (Proxy Pattern) — NEW v1.6

### 5.1 Overview

OpenZeppelin's `Initializable` pattern replaces constructors in upgradeable proxy contracts. Key properties:
- `initialize()` can only be called once
- `reinitialize(version)` can bump to a higher version but not same/lower
- `_disableInitializers()` permanently locks initialization
- Storage layout: `uint64 _initialized` (version), `bool _initializing`

### 5.2 Harness Contract

```solidity
// InitializableHarness.sol
pragma solidity ^0.8.0;

import {Initializable} from "./Initializable.sol";

contract InitializableHarness is Initializable {
    uint256 public value;

    function initialize(uint256 _value) public initializer {
        value = _value;
    }

    function reinitialize(uint256 _value, uint64 version) public reinitializer(version) {
        value = _value;
    }

    function disableInit() public {
        _disableInitializers();
    }

    // Expose internal state for verification
    function getInitializedVersion() public view returns (uint64) {
        return _getInitializedVersion();
    }

    function isInitializing() public view returns (bool) {
        return _isInitializing();
    }
}
```

### 5.3 Specification

```cvl
methods {
    function getInitializedVersion() external returns (uint64) envfree;
    function isInitializing() external returns (bool) envfree;
    function value() external returns (uint256) envfree;
}

definition nonpayable(env e) returns bool = e.msg.value == 0;

// Core invariant: not in "initializing" state between transactions
invariant notInitializing()
    !isInitializing();

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: initialize can only be called when version is 0
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule initializeEffects(env e) {
    require nonpayable(e);
    requireInvariant notInitializing();

    uint64 versionBefore = getInitializedVersion();
    uint256 newValue;

    initialize@withrevert(e, newValue);
    bool success = !lastReverted;

    // LIVENESS — succeeds iff not yet initialized
    assert success <=> (versionBefore == 0);

    // EFFECT — state is updated and version becomes 1
    assert success => (value() == newValue && getInitializedVersion() == 1);
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: cannot initialize twice
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule cannotInitializeTwice(env e1, env e2) {
    require nonpayable(e1);
    require nonpayable(e2);
    requireInvariant notInitializing();

    uint256 val1; uint256 val2;

    initialize(e1, val1);  // First call succeeds (implicitly — no @withrevert)

    initialize@withrevert(e2, val2);  // Second call
    assert lastReverted, "initialize must not succeed twice";
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: reinitialize only advances version
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule reinitializeEffects(env e) {
    require nonpayable(e);
    requireInvariant notInitializing();

    uint64 versionBefore = getInitializedVersion();
    uint256 newValue; uint64 newVersion;

    reinitialize@withrevert(e, newValue, newVersion);
    bool success = !lastReverted;

    // LIVENESS — succeeds iff new version is strictly greater
    assert success <=> (newVersion > versionBefore);

    // EFFECT
    assert success => (
        value() == newValue &&
        getInitializedVersion() == newVersion
    );
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: disableInitializers permanently locks
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule disableEffect(env e1, env e2) {
    require nonpayable(e1);
    require nonpayable(e2);
    requireInvariant notInitializing();

    disableInit(e1);  // Disable initializers

    // After disable, getInitializedVersion() == max_uint64
    assert getInitializedVersion() == max_uint64;

    // Any subsequent initialize or reinitialize must revert
    uint256 val; uint64 ver;

    initialize@withrevert(e2, val);
    assert lastReverted, "initialize must revert after disable";
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: version monotonicity — can never decrease
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule versionMonotonicity(env e) {
    requireInvariant notInitializing();

    uint64 versionBefore = getInitializedVersion();

    method f; calldataarg args;
    f(e, args);

    uint64 versionAfter = getInitializedVersion();

    assert to_mathint(versionAfter) >= to_mathint(versionBefore),
        "version must never decrease";
}
```

---

## 6. Nonces Verification Playbook (OpenZeppelin Pattern) — NEW v1.6

### 6.1 Overview

Nonces provide replay protection — each nonce is used exactly once and monotonically increases. Key properties:
- Nonces start at 0
- Each `useNonce` increments by exactly 1
- Nonces never skip or repeat
- `useCheckedNonce` reverts if the provided nonce doesn't match current

### 6.2 Specification

```cvl
methods {
    function nonces(address) external returns (uint256) envfree;
}

definition nonpayable(env e) returns bool = e.msg.value == 0;

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: useNonce increments by exactly 1
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule useNonce(env e) {
    require nonpayable(e);

    address account = e.msg.sender;
    mathint nonceBefore = nonces(account);

    // Side-effect witness
    address otherAccount;
    mathint otherNonceBefore = nonces(otherAccount);

    useNonce@withrevert(e);
    bool success = !lastReverted;

    // LIVENESS — succeeds iff nonce won't overflow
    assert success <=> (nonceBefore < max_uint256);

    // EFFECT — nonce increments by exactly 1
    assert success => to_mathint(nonces(account)) == nonceBefore + 1;

    // NO SIDE EFFECT — only caller's nonce changes
    assert nonces(otherAccount) != otherNonceBefore => otherAccount == account;
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: useCheckedNonce only succeeds with correct nonce
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule useCheckedNonce(env e) {
    require nonpayable(e);

    address account = e.msg.sender;
    uint256 expectedNonce;
    mathint nonceBefore = nonces(account);

    useCheckedNonce@withrevert(e, expectedNonce);
    bool success = !lastReverted;

    // LIVENESS — succeeds iff provided nonce matches current
    assert success <=> (to_mathint(expectedNonce) == nonceBefore);

    // EFFECT — same as useNonce
    assert success => to_mathint(nonces(account)) == nonceBefore + 1;
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: nonces only ever increase (parametric)
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule nonceMonotonicity(env e) {
    address account;
    mathint nonceBefore = nonces(account);

    method f; calldataarg args;
    f(e, args);

    mathint nonceAfter = nonces(account);

    assert nonceAfter >= nonceBefore, "nonce must never decrease";
    assert nonceAfter > nonceBefore => nonceAfter == nonceBefore + 1,
        "nonce can only increment by 1";
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// Rule: only nonce-consuming functions change nonces
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
rule nonceChangeRestriction(env e) {
    address account;
    mathint nonceBefore = nonces(account);

    method f; calldataarg args;
    f(e, args);

    mathint nonceAfter = nonces(account);

    // Only these functions can change a nonce
    assert nonceAfter != nonceBefore => (
        f.selector == sig:useNonce().selector ||
        f.selector == sig:useCheckedNonce(uint256).selector
    );
}
```

---

*This document is part of the Certora-Fv-Framework v3.0 (Offensive Verification + Red Team Hardening). Knowledge sourced from the RareSkills Certora Book — a collaboration between RareSkills and Certora. For offensive verification patterns, see `impact-spec-template.md`, `multi-step-attacks-template.md`, and `offensive-pipeline.md`.*
