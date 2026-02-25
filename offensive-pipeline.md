# Offensive Verification Pipeline

> **Framework Version:** v3.1 (Adversarial Verification Loop)  
> **Purpose:** CI/CD integration, sample configuration, counterexample triage, attack prioritization  
> **Prerequisite:** Read [impact-spec-template.md](impact-spec-template.md) and [multi-step-attacks-template.md](multi-step-attacks-template.md) before using this pipeline

---

## Overview

This document provides the **operational tooling** to run offensive verification at scale:

| Component | Purpose |
|-----------|---------|
| **Sample .conf** | Certora Prover configuration for offensive rules |
| **Pipeline Script** | Bash automation: liveness → offensive → defensive |
| **CE Severity Classification** | Triage framework for counterexamples |
| **Attack Goal Enumeration** | Systematic attack objective listing |
| **Gas Cost Analysis** | Attackability assessment based on execution cost |

---

## 1. Sample Offensive Configuration File

Save as `offensive.conf` in your project's `certora/confs/` directory.

```json
{
    "files": [
        "contracts/Vault.sol",
        "contracts/Token.sol",
        "certora/harnesses/VaultHarness.sol"
    ],
    "link": [
        "Vault:TOKEN=Token"
    ],
    "verify": "VaultHarness:certora/specs/offensive.spec",
    "msg": "Offensive Verification - Attack Surface Search",

    "// OFFENSIVE RULES — run these first": "",
    "rule": [
        "impact_hook_liveness",
        "system_value_hook_liveness",
        "attacker_cannot_profit",
        "system_value_conserved",
        "find_profitable_inputs",
        "find_max_profit_threshold"
    ],

    "// PERFORMANCE TUNING": "",
    "optimistic_loop": true,
    "optimistic_fallback": true,
    "loop_iter": 3,
    "rule_sanity": "basic",

    "// SMT SOLVER CONFIGURATION": "",
    "smt_timeout": 600,
    "prover_args": [
        "-depth 15",
        "-mediumTimeout 60",
        "-t 600"
    ],

    "// ENVIRONMENT": "",
    "solc": "solc8.26",
    "solc_evm_version": "cancun",
    "packages": [
        "@openzeppelin=lib/openzeppelin-contracts"
    ],
    "server": "production",
    "build_cache": true
}
```

### Configuration Notes

| Field | Offensive Rationale |
|-------|-------------------|
| `rule` | Lists ONLY offensive rules — defensive invariants run separately |
| `optimistic_loop` | `true` — we want the Prover to find exploits, not get stuck on loop bounds |
| `optimistic_fallback` | `true` — allows progress through unknown external calls |
| `smt_timeout` | Higher timeout (600s) gives solver more time to find deep exploits |
| `rule_sanity` | `basic` — ensures rules aren't vacuous before trusting results |
| `-depth 15` | Increases unrolling depth for complex attack paths |

### Multi-Step Attack Configuration

For rules with multiple `method` parameters (sandwich, staged), use separate conf files to avoid N³ explosion:

```json
{
    "files": ["contracts/Vault.sol", "contracts/Token.sol"],
    "verify": "VaultHarness:certora/specs/multi-step-offensive.spec",
    "msg": "Multi-Step Attacks - Flash Loan + Sandwich",
    "rule": [
        "flash_loan_attack_search",
        "flash_loan_capital_amplification",
        "sandwich_attack_search"
    ],
    "optimistic_loop": true,
    "optimistic_fallback": true,
    "loop_iter": 3,
    "smt_timeout": 900,
    "prover_args": [
        "-depth 20",
        "-mediumTimeout 120",
        "-t 900"
    ],
    "solc": "solc8.26",
    "server": "production",
    "build_cache": true
}
```

---

## 2. Offensive Verification Pipeline Script

Save as `run-offensive-pipeline.sh` in your project root.

```bash
#!/usr/bin/env bash
# ================================================================
# OFFENSIVE VERIFICATION PIPELINE
# ================================================================
# Usage: ./run-offensive-pipeline.sh [--spec-dir certora/specs] [--conf-dir certora/confs]
#
# Pipeline order (Phase 8 of Certora-Fv-Framework v3.1):
#   1. Hook Liveness Gate   — verify hooks actually capture value flows
#   2. Single-TX Attacks    — find single-call profit paths
#   3. Multi-Step Attacks   — flash loan, sandwich, staged
#   4. Defensive Invariants — prove safety only after attacks exhausted
#   5. Report               — triage and classify results
# ================================================================

set -euo pipefail

# ── Configuration ──────────────────────────────────────────────
SPEC_DIR="${1:-certora/specs}"
CONF_DIR="${2:-certora/confs}"
RESULTS_DIR="offensive-results/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULTS_DIR"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() { echo -e "${YELLOW}[$(date +%H:%M:%S)]${NC} $1"; }
pass() { echo -e "${GREEN}[PASS]${NC} $1"; }
fail() { echo -e "${RED}[FAIL]${NC} $1"; }

# ── Phase 1: Hook Liveness Gate ────────────────────────────────
log "═══ PHASE 1: Hook Liveness Gate ═══"
log "Running impact_hook_liveness and system_value_hook_liveness..."

if [[ -f "$CONF_DIR/offensive.conf" ]]; then
    certoraRun "$CONF_DIR/offensive.conf" \
        --rule impact_hook_liveness system_value_hook_liveness \
        --msg "Phase 1: Hook Liveness" \
        2>&1 | tee "$RESULTS_DIR/01-hook-liveness.log"

    # Parse results
    if grep -q "VIOLATED" "$RESULTS_DIR/01-hook-liveness.log"; then
        fail "Hook liveness VIOLATED — some functions have blind hooks."
        fail "Fix hooks before trusting any anti-invariant result."
        log "See $RESULTS_DIR/01-hook-liveness.log for details."
        # Continue but flag — don't abort, as partial coverage is informative
    else
        pass "All hooks alive — anti-invariant results can be trusted."
    fi
else
    log "WARNING: $CONF_DIR/offensive.conf not found. Creating from template..."
    log "Copy the sample .conf from offensive-pipeline.md"
fi

# ── Phase 2: Single-TX Attack Search ──────────────────────────
log "═══ PHASE 2: Single-TX Profit Search ═══"

certoraRun "$CONF_DIR/offensive.conf" \
    --rule attacker_cannot_profit system_value_conserved zero_sum_transfers \
    --msg "Phase 2: Single-TX Anti-Invariants" \
    2>&1 | tee "$RESULTS_DIR/02-single-tx-attacks.log"

if grep -q "VIOLATED" "$RESULTS_DIR/02-single-tx-attacks.log"; then
    fail "EXPLOIT FOUND — Single-TX profit path detected!"
    fail "Extract CE parameters → convert to Foundry PoC."
    echo "CRITICAL" >> "$RESULTS_DIR/severity.txt"
else
    pass "No single-TX profit path found."
fi

# ── Phase 2b: Satisfy-Based Search ────────────────────────────
log "═══ PHASE 2b: Satisfy-Based Exploit Search ═══"

certoraRun "$CONF_DIR/offensive.conf" \
    --rule find_profitable_inputs find_insolvency_path \
    --msg "Phase 2b: Satisfy Exploit Search" \
    2>&1 | tee "$RESULTS_DIR/02b-satisfy-search.log"

if grep -q "witness found" "$RESULTS_DIR/02b-satisfy-search.log" 2>/dev/null || \
   grep -q "SAT" "$RESULTS_DIR/02b-satisfy-search.log" 2>/dev/null; then
    fail "EXPLOIT WITNESS FOUND — Satisfy search produced attack parameters!"
    echo "CRITICAL" >> "$RESULTS_DIR/severity.txt"
else
    pass "No profitable inputs found via satisfy search."
fi

# ── Phase 3: Iterative Threshold ──────────────────────────────
log "═══ PHASE 3: Iterative Profit Threshold ═══"
log "Running thresholds: 0, 10^3, 10^6, 10^9, 10^12, 10^15, 10^18..."

# This requires separate spec files with different threshold values
# or use --prover_args to pass threshold as a parameter
for threshold in 0 1000 1000000 1000000000; do
    log "  Threshold: $threshold"
    certoraRun "$CONF_DIR/offensive.conf" \
        --rule find_max_profit_threshold \
        --msg "Threshold: $threshold" \
        2>&1 | tee "$RESULTS_DIR/03-threshold-$threshold.log" || true
done

# ── Phase 4: Multi-Step Attacks ───────────────────────────────
log "═══ PHASE 4: Multi-Step Attack Patterns ═══"

if [[ -f "$CONF_DIR/multi-step-offensive.conf" ]]; then
    certoraRun "$CONF_DIR/multi-step-offensive.conf" \
        --msg "Phase 4: Multi-Step Attacks" \
        2>&1 | tee "$RESULTS_DIR/04-multi-step.log"
    
    if grep -q "VIOLATED" "$RESULTS_DIR/04-multi-step.log"; then
        fail "MULTI-STEP EXPLOIT FOUND!"
        echo "CRITICAL" >> "$RESULTS_DIR/severity.txt"
    else
        pass "No multi-step exploit path found."
    fi
else
    log "Skipping multi-step (no multi-step-offensive.conf found)."
fi

# ── Phase 5: Defensive Invariants ─────────────────────────────
log "═══ PHASE 5: Defensive Verification ═══"

if [[ -f "$CONF_DIR/defensive.conf" ]]; then
    certoraRun "$CONF_DIR/defensive.conf" \
        --msg "Phase 5: Defensive Invariants" \
        2>&1 | tee "$RESULTS_DIR/05-defensive.log"
    
    if grep -q "VIOLATED" "$RESULTS_DIR/05-defensive.log"; then
        fail "Defensive invariant VIOLATED — safety property broken."
        echo "HIGH" >> "$RESULTS_DIR/severity.txt"
    else
        pass "All defensive invariants hold."
    fi
else
    log "Skipping defensive (no defensive.conf found)."
fi

# ── Summary ───────────────────────────────────────────────────
log "═══ PIPELINE COMPLETE ═══"
log "Results: $RESULTS_DIR/"
echo ""

if [[ -f "$RESULTS_DIR/severity.txt" ]]; then
    CRIT_COUNT=$(grep -c "CRITICAL" "$RESULTS_DIR/severity.txt" 2>/dev/null || echo 0)
    HIGH_COUNT=$(grep -c "HIGH" "$RESULTS_DIR/severity.txt" 2>/dev/null || echo 0)
    echo "========================================"
    echo "FINDINGS SUMMARY"
    echo "========================================"
    echo "CRITICAL: $CRIT_COUNT"
    echo "HIGH:     $HIGH_COUNT"
    echo "========================================"
    echo ""
    fail "⚠ EXPLOITS DETECTED — Convert CEs to Foundry PoCs."
else
    pass "No exploits detected across all attack patterns."
    log "Protocol appears safe within modeled scope."
fi
```

### Pipeline Integration with CI/CD

#### GitHub Actions

```yaml
# .github/workflows/offensive-verification.yml
name: Offensive Verification

on:
  push:
    branches: [main, develop]
  pull_request:
    paths:
      - 'contracts/**'
      - 'certora/**'

jobs:
  offensive-verification:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Certora CLI
        run: pip install certora-cli

      - name: Set Certora Key
        env:
          CERTORAKEY: ${{ secrets.CERTORAKEY }}
        run: echo "CERTORAKEY=$CERTORAKEY" >> $GITHUB_ENV

      - name: Run Offensive Pipeline
        run: |
          chmod +x run-offensive-pipeline.sh
          ./run-offensive-pipeline.sh certora/specs certora/confs

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: offensive-results
          path: offensive-results/
```

#### Pre-Commit Hook

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push — run quick offensive checks before push
echo "Running quick offensive smoke test..."
certoraRun certora/confs/offensive.conf \
    --rule impact_hook_liveness attacker_cannot_profit \
    --smt_timeout 120 \
    --msg "Pre-push smoke test"
```

---

## 3. Counterexample Severity Classification

When the Prover produces a counterexample (VIOLATED result), classify it:

### Severity Matrix

| Severity | Criteria | Action |
|----------|----------|--------|
| **CRITICAL** | `attacker_cannot_profit` VIOLATED with `profit > 10^15` | Immediate PoC, halt other work |
| **CRITICAL** | `system_value_conserved` VIOLATED | Immediate investigation |
| **HIGH** | `attacker_cannot_profit` VIOLATED with `profit > 0` | PoC within 24h |
| **HIGH** | Multi-step exploit found (flash loan / sandwich) | PoC + root cause analysis |
| **MEDIUM** | `find_profitable_inputs` SAT but `attacker_cannot_profit` VERIFIED | Profit requires preconditions — investigate constraints |
| **MEDIUM** | `zero_sum_transfers` VIOLATED | Value accounting error — may or may not be exploitable |
| **LOW** | `find_insolvency_path` SAT with extreme parameters | Edge case — document, may not be practical |
| **INFO** | Hook liveness VIOLATED for admin-only functions | Expected — admin functions don't need attack coverage |
| **FALSE POS** | CE uses `msg.sender == address(0)` or other impossible state | Over-approximation — tighten `require` constraints |

### CE Triage Decision Tree

```
CE received from Prover
│
├── Is msg.sender == address(0)?
│   └── YES → FALSE POSITIVE. Add: require e.msg.sender != 0;
│
├── Does CE use impossible storage state?
│   └── YES → FALSE POSITIVE. Add: requireInvariant for missing constraint.
│
├── Is the function admin-only?
│   └── YES → Check if attacker can become admin. If not → LOW/INFO.
│
├── Does profit > 0?
│   ├── YES, profit > 10^15 → CRITICAL
│   ├── YES, profit > 0 but < 10^6 → HIGH (may be dust, still investigate)
│   └── NO → Check if system value changed → MEDIUM
│
├── Is it a multi-step CE?
│   ├── Flash loan in single block → CRITICAL
│   ├── Multi-block staged → HIGH (slower but still exploitable)
│   └── Requires > 100 blocks → MEDIUM (may not be practical)
│
└── Convert to Foundry PoC
    ├── PoC reproduces on fork → CONFIRMED EXPLOIT
    ├── PoC fails on fork → Check model completeness
    └── PoC requires unrealistic gas → Document as theoretical
```

### CE Parameter Extraction Checklist

When extracting exploit parameters from a Certora counterexample:

| CE Field | Maps To | Foundry PoC Element |
|----------|---------|-------------------|
| `e.msg.sender` | Attacker address | `vm.prank(attacker)` |
| `e.msg.value` | ETH sent | `target.f{value: amount}()` |
| `e.block.number` | Block context | `vm.roll(blockNum)` |
| `e.block.timestamp` | Time context | `vm.warp(timestamp)` |
| Function called (`f`) | Entry point | `target.vulnerableFunction()` |
| `calldataarg` values | Attack parameters | Function arguments |
| `actor_value[attacker]` before | Initial balance | `token.balanceOf(attacker)` before |
| `actor_value[attacker]` after | Final balance | `token.balanceOf(attacker)` after |
| `total_system_value` delta | Protocol loss | `token.balanceOf(address(vault))` delta |
| Storage slot changes | State manipulation | Check in call trace |

---

## 4. Attack Goal Enumeration & Prioritization

### Systematic Attack Objectives

For any DeFi protocol, enumerate attack goals in priority order:

| Priority | Attack Objective | CVL Rule Pattern | Economic Class |
|----------|-----------------|------------------|---------------|
| **P0** | Drain protocol funds | `assert actor_value[attacker] <= initial` | Direct theft |
| **P0** | Mint unbacked tokens | `assert totalSupply_after <= totalSupply_before + minted` | Inflation |
| **P1** | Manipulate share price | `assert sharePrice_after == sharePrice_before` (for non-deposit/withdraw) | Dilution |
| **P1** | Extract via flash loan | `flash_loan_attack_search` | Amplified extraction |
| **P1** | Sandwich for profit | `sandwich_attack_search` | MEV extraction |
| **P2** | Force insolvency | `satisfy insolvent_state` | Protocol failure |
| **P2** | Freeze withdrawals | `satisfy liquidity_frozen` | DoS |
| **P2** | Socialize losses to users | `assert socialized_loss[user] == 0` | Loss distribution |
| **P3** | Governance takeover | `governance_extraction_check` | Control hijack |
| **P3** | Oracle manipulation | `price_manipulation_sandwich` | Price attack |
| **P3** | Accumulate-then-extract | `staged_attack_accumulation` | Gradual theft |

### Protocol-Specific Attack Surface

Map your protocol type to relevant attack goals:

| Protocol Type | Primary Attack Goals | Secondary Attack Goals |
|---------------|---------------------|----------------------|
| **Vault/ERC-4626** | Share dilution, deposit/withdraw arbitrage | First depositor attack, rounding exploits |
| **Lending** | Bad debt creation, liquidation manipulation | Interest rate gaming, collateral factor abuse |
| **AMM/DEX** | Price manipulation, sandwich, impermanent loss amplification | LP token value extraction, fee theft |
| **Staking** | Reward inflation, slash avoidance | Double-claiming, unstaking manipulation |
| **Governance** | Flash loan voting, proposal manipulation | Timelock bypass, quorum gaming |
| **Bridge** | Message replay, double-spend | Finality manipulation, relayer manipulation |
| **Perpetuals** | Funding rate manipulation, liquidation hunting | Position size exploitation, mark price deviation |

### Ranking Function for Counterexamples

When multiple CEs are found, rank by exploitability:

```
Rank Score = Impact × Likelihood × (1 / Cost)

Where:
  Impact     = attacker_profit (from CE, in token units)
  Likelihood = 1.0 (single-tx) | 0.8 (same-block multi-tx) | 0.5 (multi-block)
  Cost       = estimated gas cost × current gas price
```

| Score Range | Classification | Action |
|-------------|---------------|--------|
| > 10 ETH equivalent | CRITICAL | Immediate escalation |
| 1-10 ETH | HIGH | Priority PoC |
| 0.01-1 ETH | MEDIUM | Investigate feasibility |
| < 0.01 ETH | LOW | Document, may not be economical |

---

## 5. Economic Assertion Templates

Pre-built assertion patterns for common attack objectives:

### Token Drain Detection
```cvl
// No single function should allow extracting more than deposited
rule no_excess_withdrawal(env e, uint256 amount) {
    require valid_sender(e);
    address user = e.msg.sender;

    mathint balance_before = getBalance(e, user);
    mathint vault_before = getVaultTotal(e);

    withdraw(e, amount);

    mathint balance_after = getBalance(e, user);
    mathint vault_after = getVaultTotal(e);
    mathint withdrawn = balance_after - balance_before;

    assert withdrawn <= balance_before,
        "DRAIN: User withdrew more than their balance";
    assert vault_before - vault_after <= to_mathint(amount),
        "DRAIN: Vault lost more than withdrawal amount";
}
```

### Share Dilution Detection
```cvl
// Share price should not decrease from a single operation
// (except for legitimate fee/loss events)
rule share_price_monotonic(env e, method f)
    filtered { f -> !f.isView && !f.isFallback }
{
    require valid_sender(e);

    mathint total_assets_before = totalAssets(e);
    mathint total_shares_before = totalSupply(e);
    require total_shares_before > 0;

    // price_before = total_assets / total_shares (in mathint)
    mathint price_numerator_before = total_assets_before * 10^18;

    calldataarg args;
    f(e, args);

    mathint total_assets_after = totalAssets(e);
    mathint total_shares_after = totalSupply(e);
    require total_shares_after > 0;

    mathint price_numerator_after = total_assets_after * 10^18;

    // Share price should not decrease
    assert price_numerator_after * total_shares_before >=
           price_numerator_before * total_shares_after,
        "DILUTION: Share price decreased — attacker may be diluting other holders";
}
```

### First Depositor Attack Detection
```cvl
// Classic ERC-4626 attack: first depositor inflates share price
rule first_depositor_safe(env e_attacker, env e_victim) {
    require valid_sender(e_attacker);
    require valid_sender(e_victim);
    require e_attacker.msg.sender != e_victim.msg.sender;

    // Vault is empty
    require totalSupply(e_attacker) == 0;
    require totalAssets(e_attacker) == 0;

    // Attacker deposits small amount
    uint256 attacker_deposit;
    require attacker_deposit > 0 && attacker_deposit <= 10;
    deposit(e_attacker, attacker_deposit);

    // Attacker donates large amount directly to vault
    // (Modeled as direct token transfer — adapt to your protocol)
    // In practice, this would need a hook on the vault's token balance

    // Victim deposits normally
    uint256 victim_deposit;
    require victim_deposit > 0;
    deposit(e_victim, victim_deposit);

    // Victim should get shares proportional to their deposit
    mathint victim_shares = balanceOf(e_victim, e_victim.msg.sender);

    assert victim_shares > 0,
        "FIRST DEPOSITOR ATTACK: Victim received 0 shares";
}
```

---

## 6. Gas Cost Analysis

### Attack Feasibility Assessment

Not all counterexamples are practical — gas cost matters:

| Gas Used | Cost at 30 gwei | Cost at 100 gwei | Feasibility |
|----------|-----------------|-------------------|-------------|
| 100K | 0.003 ETH | 0.01 ETH | Trivial |
| 500K | 0.015 ETH | 0.05 ETH | Easy |
| 1M | 0.03 ETH | 0.1 ETH | Feasible |
| 5M | 0.15 ETH | 0.5 ETH | Moderate cost |
| 15M | 0.45 ETH | 1.5 ETH | Block limit concern |
| 30M | 0.9 ETH | 3.0 ETH | At EVM block gas limit |

### Profitability Threshold

An exploit is **economically viable** when:

$$\text{profit} > \text{gas cost} + \text{opportunity cost} + \text{risk premium}$$

For automated/MEV attacks:
- Gas cost ≈ gas_used × gas_price
- Opportunity cost ≈ 0 (automated)
- Risk premium ≈ 0 (atomic, no capital at risk if using flash loans)

For multi-block attacks:
- Add capital lockup cost (DeFi lending rate × duration × amount)
- Add execution risk premium (competing attackers, chain reorgs)

### Foundry Gas Estimation

After converting CE to PoC, measure actual gas:

```solidity
function testExploitGasCost() public {
    uint256 gasStart = gasleft();

    // ... exploit steps ...

    uint256 gasUsed = gasStart - gasleft();
    uint256 profitWei = token.balanceOf(attacker) - initialBalance;
    uint256 gasCostWei = gasUsed * 30 gwei; // 30 gwei estimate

    console2.log("Gas used:", gasUsed);
    console2.log("Profit (ETH):", profitWei / 1e18);
    console2.log("Gas cost (ETH):", gasCostWei / 1e18);
    console2.log("Net profit (ETH):", (profitWei - gasCostWei) / 1e18);

    // Only flag if economically viable
    assertGt(profitWei, gasCostWei, "Exploit not economically viable");
}
```

---

## 7. Harness Templates for Rapid Bootstrapping

### Minimal Offensive Harness

A harness exposes internal state for the Prover:

```solidity
// certora/harnesses/VaultHarness.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "../../contracts/Vault.sol";

contract VaultHarness is Vault {
    // Expose internal state for Prover
    function getInternalBalance(address user) external view returns (uint256) {
        return _balances[user];
    }

    function getInternalTotalAssets() external view returns (uint256) {
        return _totalAssets;
    }

    function getSharePrice() external view returns (uint256) {
        if (totalSupply() == 0) return 0;
        return (_totalAssets * 1e18) / totalSupply();
    }

    // Expose protocol-specific internal state
    // Add getters for any private variable the spec needs to track
}
```

### Using Declaration Pattern

```cvl
// In your offensive.spec
using VaultHarness as vault;
using Token as token;

methods {
    // Harness getters (envfree = no env needed)
    function vault.getInternalBalance(address) external returns (uint256) envfree;
    function vault.getInternalTotalAssets() external returns (uint256) envfree;
    function vault.getSharePrice() external returns (uint256) envfree;

    // External token (summarize if unknown implementation)
    function token.balanceOf(address) external returns (uint256) envfree;
    function token.totalSupply() external returns (uint256) envfree;

    // Catch-all for unknown external calls
    function _._ external => DISPATCH [
        token.transfer(address, uint256),
        token.transferFrom(address, address, uint256)
    ] default NONDET;
}
```

---

## 8. Quick-Start Checklist

When starting offensive verification on a new protocol:

### Day 1: Setup (2-4 hours)
- [ ] Read protocol documentation and identify value flows
- [ ] List all assets: tokens, ETH, shares, LP tokens, internal balances
- [ ] Create harness exposing internal state
- [ ] Copy `offensive.conf` template and adapt
- [ ] Copy impact ghosts and hooks from `impact-spec-template.md`
- [ ] Adapt hooks to protocol's storage layout

### Day 2: Liveness + Single-TX (4-6 hours)
- [ ] Run `impact_hook_liveness` — fix any blind hooks
- [ ] Run `system_value_hook_liveness` — fix any missing value tracking
- [ ] Complete Value Flow Completeness Checklist
- [ ] Run `attacker_cannot_profit` — investigate any CEs
- [ ] Run `find_profitable_inputs` — check for satisfy witnesses
- [ ] Begin iterative threshold protocol if profits found

### Day 3: Multi-Step + PoC (4-6 hours)
- [ ] Copy relevant multi-step patterns (flash loan, sandwich, staged)
- [ ] Adapt to protocol-specific function calls
- [ ] Add `filtered` clauses to bound search space
- [ ] Run multi-step rules
- [ ] Convert any CEs to Foundry PoCs (use `poc-template-foundry.md`)
- [ ] Measure gas cost and confirm economic viability

### Day 4: Final Defensive Proof + Report (2-4 hours)
- [ ] Review feedback loop convergence (SAT/UNSAT results for all offensive specs)
- [ ] Write full defensive spec (refined by offensive findings)
- [ ] Run final defensive proof (ALWAYS LAST)
- [ ] Classify all CEs using severity matrix
- [ ] Write security findings using `vulnerability-report-template.md`
- [ ] Document attack surfaces that were proven safe

---

## Integration Checklist

- [ ] Created `offensive.conf` adapted to my protocol
- [ ] Created `multi-step-offensive.conf` for multi-step rules
- [ ] Created harness exposing internal state
- [ ] `run-offensive-pipeline.sh` runs without errors
- [ ] All hook liveness checks pass (or blind spots documented)
- [ ] All CEs classified using severity matrix
- [ ] Economic viability assessed for all CEs
- [ ] Foundry PoCs created for CRITICAL/HIGH findings
- [ ] Results archived in `offensive-results/` directory
- [ ] Findings documented in security report
