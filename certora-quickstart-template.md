# CERTORA QUICKSTART TEMPLATE

> **Use this template to apply the Certora workflow to ANY contract**  
> **Copy this file and fill in the blanks for each new verification project**  
> **Version:** 1.9 (Red Team Hardening)

---

## WHAT'S NEW IN v1.9

- **Failure-path reachability:** `satisfy lastReverted` rules validate revert paths are reachable before biconditional `<=>` rules
- **Custom summary accuracy:** Mandatory annotation (Exact/Over/Under) and justification for every custom summary
- **Invariant dependency DAG:** `@dev Level: N` annotations, cycle detection protocol, level-by-level proving
- **Satisfy liveness annotation:** Explicit warning that `satisfy !lastReverted` proves liveness, not effect

### Previous Enhancements
- **v1.8:** Reachability validation (`satisfy` rules as mandatory pre-step)
- **v1.7:** Prover v8.8.0 builtin rules (`uncheckedOverflow`, `safeCasting`)
- **v1.6:** Revert/failure-path coverage (`@withrevert`, biconditional `<=>`, MUST REVERT WHEN)
- **v1.5:** RareSkills integration, CVL Deep Dive, Verification Playbooks
- **v1.4:** Advanced CLI Reference, Performance optimization, PoC templates
- **v1.3:** Property Prioritization, Dual Mindset, Test Mining, 5-step CE Investigation
- **v1.2:** Dual Mindset approach, Test Mining for property discovery
- **v1.1:** Validation-to-Real-Spec transition, Chat Prompts

---

## TABLE OF CONTENTS

1. [Initial Setup](#1-initial-setup)
2. [Phase 0: Contract Analysis](#2-phase-0-contract-analysis)
3. [Phase 2: Property Discovery](#3-phase-2-property-discovery)
4. [Phase 2.5: Classification](#4-phase-25-classification)
5. [Phase 3.5: Causal Validation](#5-phase-35-causal-validation)
6. [Phase 4-6: Modeling & Sanity](#6-phase-4-6-modeling--sanity)
7. [Phase 7: Write CVL](#7-phase-7-write-cvl)
8. [Running & Debugging](#8-running--debugging)
9. [Quick Reference Commands](#9-quick-reference-commands)

---

## 1. INITIAL SETUP

### Step 1.1: Create Folder Structure

```bash
# Replace CONTRACT_NAME with your contract (e.g., Escrow, Timelock, Vault)
CONTRACT_NAME="YourContract"

# Create spec authoring workspace
mkdir -p spec_authoring

# Create the 4 required files
touch spec_authoring/${CONTRACT_NAME,,}_spec_authoring.md
touch spec_authoring/${CONTRACT_NAME,,}_candidate_properties.md
touch spec_authoring/${CONTRACT_NAME,,}_causal_validation.md

# Create certora directories if they don't exist
mkdir -p certora/specs
mkdir -p certora/confs
mkdir -p certora/harnesses
mkdir -p certora/helpers
```

### Step 1.2: File Purpose Reference

| File | Purpose | When to Use |
|------|---------|-------------|
| `spec_authoring/{contract}_spec_authoring.md` | Main analysis workspace | Phases 0, -1, 3, 4, 5, 6 |
| `spec_authoring/{contract}_candidate_properties.md` | Plain English properties | Phase 2 |
| `spec_authoring/{contract}_causal_validation.md` | Mutation path analysis | Phase 3.5 |
| `certora/specs/validation_{contract}.spec` | Validation rules (run first) | Phase 3.5 |
| `certora/specs/{Contract}.spec` | Actual verification spec | Phase 7 |
| `certora/confs/validation_{contract}.conf` | Config for validation | Phase 3.5 |
| `certora/confs/{Contract}.conf` | Config for real spec | Phase 7 |

### Step 1.3: Document Templates

Copy these templates into your files:

---

## 2. PHASE 0: CONTRACT ANALYSIS

### Step 2.1: Open the Contract

```bash
# Open the contract you're verifying
code contracts/YourContract.sol
```

### Step 2.2: Fill in `{contract}_spec_authoring.md`

Use this template:

```markdown
# {CONTRACT_NAME} Specification Authoring Workspace

## Phase 0: Verification Scope

### 0.1 Verification Boundary

**Primary Contract:** `{ContractName}.sol`

**In-Scope Contracts:**
| Contract | Role | Why In-Scope |
|----------|------|--------------|
| {ContractName} | Main | Primary verification target |
| | | |

**Out-of-Scope:**
| Contract | Why Out-of-Scope |
|----------|------------------|
| | |

### 0.2 Entry Points (State-Changing Functions)

| # | Function | Visibility | State Changes | External Calls |
|---|----------|------------|---------------|----------------|
| 1 | | | | |
| 2 | | | | |

**Command to find entry points:**
```bash
grep -n "function.*external\|function.*public" contracts/YourContract.sol
```

### 0.3 View Functions

| Function | Returns | Used By |
|----------|---------|---------|
| | | |

### 0.4 State Mutation Map

| Storage Variable | Type | Modified By Functions |
|------------------|------|----------------------|
| | | |

**Command to find storage variables:**
```bash
grep -n "^\s*mapping\|^\s*uint\|^\s*address\|^\s*bool\|^\s*struct" contracts/YourContract.sol | head -50
```

---

## Phase -1: Trust Boundary Analysis

### -1.1 External Contracts Called

| External Contract | Functions Called | Trust Level |
|-------------------|------------------|-------------|
| | | |

### -1.2 Interaction Ownership Table

| Data | Owner Contract | {ContractName} Access |
|------|----------------|----------------------|
| | | Read / Write / Both |

### -1.3 Modeling Obligations

| External Contract | Modeling Strategy | Justification |
|-------------------|-------------------|---------------|
| | DISPATCHER / NONDET / HAVOC | |

---

## Phase 3: State Classification

### Trusted State (Owned by {ContractName})
- 

### Untrusted State (External)
- 

---

## Phase 4: Modeling Decisions

### NONDET Summaries (Justified)
| Function | Why NONDET is Safe |
|----------|-------------------|
| | |

### DISPATCHER Summaries
| Function | Dispatches To |
|----------|---------------|
| | |

---

## Phase 5: Ghost Requirements

### Ghosts Needed
| Ghost | Type | Purpose | Hook On |
|-------|------|---------|---------|
| | | | |

### Ghosts NOT Needed
| Considered | Why Not Needed |
|------------|----------------|
| | |

---

## Phase 6: Sanity Gate Checklist

- [ ] All entry points enumerated
- [ ] All external reads have modeling decision
- [ ] All external writes have modeling decision
- [ ] Array parameters bounded
- [ ] Timestamp ranges bounded
- [ ] No ghost mirrors owned storage
- [ ] All ghosts have init_state axiom
- [ ] All hooks use correct types
```

### Step 2.3: Commands to Gather Information

```bash
# Find all external/public functions
grep -n "function.*external\|function.*public" contracts/YourContract.sol

# Find all storage variables
grep -n "^\s*mapping\|^\s*uint\|^\s*address\|^\s*bool" contracts/YourContract.sol

# Find all external calls
grep -n "\.[a-zA-Z]*(" contracts/YourContract.sol | grep -v "//"

# Find all imports
grep -n "^import" contracts/YourContract.sol

# Find all events (state changes often emit events)
grep -n "emit " contracts/YourContract.sol
```

---

## 3. PHASE 2: PROPERTY DISCOVERY

### Step 3.1: Fill in `{contract}_candidate_properties.md`

Use this template:

```markdown
# {CONTRACT_NAME} Candidate Properties

## Category A: Asset Safety (Solvency)

### A1. {Property Name}
**Plain English:** 
**Impact if Violated:** 
**Variables Involved:** 
**External Truths Needed:** 

### A2. ...

---

## Category B: Functional Correctness

### B1. {Property Name}
**Plain English:** 
**Trigger:** When {function} is called...
**Expected Outcome:** 
**Variables Involved:** 

---

## Category C: State Consistency

### C1. {Property Name}
**Plain English:** 
**Relationship:** {variable1} must equal/match {variable2}
**Variables Involved:** 

---

## Category D: Access Control

### D1. {Property Name}
**Plain English:** Only {role} can call {function}
**Protected Functions:** 
**Authorized Callers:** 

---

## Category E: State Machine

### E1. {Property Name}
**Plain English:** State can only transition from X to Y
**Valid Transitions:** 
**Invalid Transitions:** 
```

### Step 3.2: Property Discovery Questions

Ask yourself:

1. **Solvency:** "What would make users lose funds?"
2. **Correctness:** "What should happen when X is called?"
3. **Consistency:** "What relationships must always hold?"
4. **Access:** "Who should be able to call what?"
5. **State:** "What are the valid state transitions?"

---

## 4. PHASE 2.5: CLASSIFICATION

### Step 4.1: Apply Decision Flowchart

For each property in your candidate list:

```
┌─────────────────────────────────────────┐
│ Q0: Is all relevant state modeled?      │
│     (No external reads without model)   │
└─────────────────────────────────────────┘
         │
         ▼ YES
┌─────────────────────────────────────────┐
│ Q1: Is it about "Can X EVER happen?"    │
│     vs "WHEN Y, does Z happen?"         │
└─────────────────────────────────────────┘
         │
    ┌────┴────┐
    ▼ EVER   ▼ WHEN
INVARIANT    RULE
```

### Step 4.2: Add Classification to Properties

```markdown
### A1. ETH Solvency
**Plain English:** Contract always has enough ETH to pay all users
**Classification:** INVARIANT
**Reasoning:** "Can the contract EVER owe more than it has?" = EVER question
```

---

## 5. PHASE 3.5: CAUSAL VALIDATION

### Step 5.1: Fill in `{contract}_causal_validation.md`

```markdown
# {CONTRACT_NAME} Causal Validation

## Purpose
Enumerate ALL mutation paths for each INVARIANT variable.
If we miss one, the invariant will have a false counterexample.

---

## Invariant: {Name}

### Variables in Invariant
1. `{variable1}` - {description}
2. `{variable2}` - {description}

### Mutation Paths for `{variable1}`

| # | Function | Change Type | Notes |
|---|----------|-------------|-------|
| 1 | | increase/decrease | |
| 2 | | | |

### Mutation Paths for `{variable2}`

| # | Function | Change Type | Notes |
|---|----------|-------------|-------|
| 1 | | | |

### Validation Rule

```cvl
rule validation_mutation_paths_{variable}(method f)
    filtered { f -> f.contract == currentContract && !f.isView }
{
    uint256 before = getVariable();
    
    env e;
    calldataarg args;
    f(e, args);
    
    uint256 after = getVariable();
    
    assert before != after => (
        f.selector == sig:function1().selector ||
        f.selector == sig:function2().selector
    ), "Unmodeled mutation path!";
}
```
```

### Step 5.2: Create `validation_{contract}.spec`

```bash
touch certora/specs/validation_{contract}.spec
```

Template:

```cvl
/*
 * {CONTRACT_NAME} CAUSAL VALIDATION SPEC
 * Run this FIRST. All rules must PASS before writing real spec.
 */

using {Contract} as currentContract;

methods {
    // List all state-changing functions
    function function1() external;
    function function2(uint256) external;
    
    // External contracts - use DISPATCHER or NONDET
    function _.externalCall() external => DISPATCHER(true);
}

// ═══════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ═══════════════════════════════════════════════════════════════

function getVariable1() returns uint256 {
    return currentContract.variable1;
}

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULES
// ═══════════════════════════════════════════════════════════════

rule validation_mutation_paths_variable1(method f)
    filtered { f -> f.contract == currentContract && !f.isView }
{
    uint256 before = getVariable1();
    
    env e;
    calldataarg args;
    f(e, args);
    
    uint256 after = getVariable1();
    
    assert before != after => (
        f.selector == sig:function1().selector ||
        f.selector == sig:function2(uint256).selector
    ), "CAUSAL VIOLATION: Unmodeled mutation path";
}

// Add more validation rules for each variable...
```

### Step 5.3: Create `validation_{contract}.conf`

```bash
touch certora/confs/validation_{contract}.conf
```

Template:

```json
{
    "files": [
        "contracts/YourContract.sol",
        "contracts/Dependency1.sol",
        "certora/harnesses/DummyToken.sol"
    ],
    "link": [
        "YourContract:TOKEN=DummyToken"
    ],
    "msg": "YourContract Causal Validation",
    "packages": [
        "@openzeppelin=lib/openzeppelin-contracts"
    ],
    "solc_evm_version": "cancun",
    "solc": "solc",
    "optimistic_loop": true,
    "optimistic_fallback": true,
    "loop_iter": "3",
    "rule_sanity": "basic",
    "build_cache": true,
    "server": "production",
    "verify": "YourContract:certora/specs/validation_{contract}.spec"
}
```

### Step 5.4: Run Validation

```bash
# Clear cache and run
rm -rf .certora_internal
certoraRun certora/confs/validation_{contract}.conf
```

**Expected Results:**
- ✅ All rules PASS → Proceed to Phase 7
- ❌ Any rule FAILS → You missed a mutation path, fix and re-run

---

## 6. PHASE 4-6: MODELING & SANITY

### Step 6.1: Modeling Decisions Checklist

For each external contract:

| Decision | When to Use | Example |
|----------|-------------|---------|
| `DISPATCHER(true)` | You have the implementation | Token transfers |
| `NONDET` | Result doesn't affect invariant | Logging calls |
| `HAVOC_ALL` | External state could change anything | Unknown contracts |

### Step 6.2: Ghost Checklist

| Need Ghost? | Situation |
|-------------|-----------|
| ✅ YES | Sum of mapping values |
| ✅ YES | Historical value tracking |
| ❌ NO | Storage you can read directly |
| ❌ NO | Single value in storage |

### Step 6.3: Sanity Gate (MUST COMPLETE)

```markdown
## Sanity Gate Checklist

### Entry Points
- [ ] Listed ALL external functions
- [ ] Listed ALL public functions
- [ ] Checked for receive()/fallback()

### External Calls
- [ ] Every external call has modeling decision
- [ ] NONDET calls justified (don't affect invariant)
- [ ] DISPATCHER calls have implementation linked

### Bounds
- [ ] Array parameters: `require arr.length < 100;`
- [ ] Timestamps: `require e.block.timestamp < 2^40;`
- [ ] Balances: `require balance < 2^128;`

### Ghosts
- [ ] Every ghost has `init_state axiom`
- [ ] Hook types match Solidity types exactly
- [ ] No ghost mirrors directly-readable storage
```

---

## 7. PHASE 7: WRITE CVL

### Step 7.1: Create Real Spec

```bash
touch certora/specs/{Contract}.spec
```

### Step 7.2: Spec Structure Template

```cvl
/*
 * ═══════════════════════════════════════════════════════════════
 * {CONTRACT_NAME} VERIFICATION SPEC
 * ═══════════════════════════════════════════════════════════════
 */

// ═══════════════════════════════════════════════════════════════
// IMPORTS & USING
// ═══════════════════════════════════════════════════════════════

using DummyToken as token;
using YourContract as currentContract;

// ═══════════════════════════════════════════════════════════════
// METHODS BLOCK
// ═══════════════════════════════════════════════════════════════

methods {
    // Contract functions
    function deposit(uint256) external;
    function withdraw(uint256) external;
    
    // View functions (envfree)
    function balanceOf(address) external returns (uint256) envfree;
    
    // External contracts
    function _.transfer(address,uint256) external => DISPATCHER(true);
    function _.balanceOf(address) external => DISPATCHER(true);
    
    // Unrelated calls
    function _.someLoggingFunction() external => NONDET;
}

// ═══════════════════════════════════════════════════════════════
// GHOSTS
// ═══════════════════════════════════════════════════════════════

ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

// ═══════════════════════════════════════════════════════════════
// HOOKS
// ═══════════════════════════════════════════════════════════════

hook Sstore currentContract.balances[KEY address user] uint256 newBal 
    (uint256 oldBal) {
    sumBalances = sumBalances + newBal - oldBal;
}

// ═══════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ═══════════════════════════════════════════════════════════════

function validState() {
    requireInvariant solvency();
    requireInvariant stateValid();
}

// ═══════════════════════════════════════════════════════════════
// INVARIANTS (ordered by dependency)
// ═══════════════════════════════════════════════════════════════

/// @title State enum is always valid
invariant stateValid()
    getState() <= 2

/// @title Contract is always solvent
invariant solvency()
    nativeBalances[currentContract] >= getTotalOwed()
    {
        preserved {
            requireInvariant stateValid();
        }
    }

// ═══════════════════════════════════════════════════════════════
// RULES
// ═══════════════════════════════════════════════════════════════

/// @title Deposit increases user balance
rule deposit_increasesBalance(address user, uint256 amount) {
    validState();
    
    uint256 balBefore = balanceOf(user);
    
    env e;
    require e.msg.sender == user;
    deposit(e, amount);
    
    uint256 balAfter = balanceOf(user);
    
    assert balAfter >= balBefore;
}
```

### Step 7.3: Create Real Config

```bash
touch certora/confs/{Contract}.conf
```

---

## 8. RUNNING & DEBUGGING

### Step 8.1: Run Validation First (ALWAYS)

```bash
certoraRun certora/confs/validation_{contract}.conf
```

### Step 8.2: Run Real Spec

```bash
certoraRun certora/confs/{Contract}.conf
```

### Step 8.3: Debug Compilation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `X is not a valid EVM type` | Enum/custom type in hook | Use underlying type or Solidity type |
| `already declared in scope` | Name conflicts with contract | Rename CVL function |
| `could not find method` | Wrong function signature | Check contract for exact signature |
| `Type mismatch` | Hook type doesn't match storage | Use exact Solidity type |

### Step 8.4: Debug Counterexamples

Use `certora-ce-diagnosis-framework.md`:

1. **Check if CE is real or spurious**
2. **If spurious:** Add `require` or fix modeling
3. **If real:** You found a bug!

---

## 9. QUICK REFERENCE COMMANDS

### Setup Commands

```bash
# Create folder structure
mkdir -p spec_authoring certora/{specs,confs,harnesses,helpers}

# Create analysis files
touch spec_authoring/{contract}_spec_authoring.md
touch spec_authoring/{contract}_candidate_properties.md
touch spec_authoring/{contract}_causal_validation.md

# Create spec files
touch certora/specs/validation_{contract}.spec
touch certora/specs/{Contract}.spec
touch certora/confs/validation_{contract}.conf
touch certora/confs/{Contract}.conf
```

### Analysis Commands

```bash
# Find entry points
grep -n "function.*external\|function.*public" contracts/YourContract.sol

# Find storage variables
grep -n "^\s*mapping\|^\s*uint\|^\s*address" contracts/YourContract.sol

# Find external calls
grep -n "\.[a-zA-Z]*(" contracts/YourContract.sol | grep -v "//"

# Find events
grep -n "emit " contracts/YourContract.sol
```

### Run Commands

```bash
# Clear cache
rm -rf .certora_internal

# Run validation
certoraRun certora/confs/validation_{contract}.conf

# Run with specific rules
certoraRun certora/confs/{Contract}.conf --rule "ruleName"

# Run with verbose output
certoraRun certora/confs/{Contract}.conf 2>&1 | tee output.log
```

### Debug Commands

```bash
# Check for type issues
grep -n "hook Sstore" certora/specs/*.spec

# Check function signatures in contract
grep -n "function " contracts/YourContract.sol

# Verify solc version
solc --version
```

---

## WORKFLOW SUMMARY

```
┌─────────────────────────────────────────────────────────────────┐
│                    CERTORA WORKFLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CREATE FILES                                                 │
│     └─► spec_authoring/{contract}_*.md                          │
│     └─► certora/specs/validation_{contract}.spec                │
│                                                                  │
│  2. ANALYZE CONTRACT (Phase 0, -1)                              │
│     └─► Fill {contract}_spec_authoring.md                       │
│     └─► Entry points, external calls, storage                   │
│                                                                  │
│  3. DISCOVER PROPERTIES (Phase 2)                               │
│     └─► Fill {contract}_candidate_properties.md                 │
│     └─► Plain English only!                                     │
│                                                                  │
│  4. CLASSIFY (Phase 2.5)                                        │
│     └─► INVARIANT vs RULE decision                              │
│                                                                  │
│  5. VALIDATE CAUSALLY (Phase 3.5)                               │
│     └─► Fill {contract}_causal_validation.md                    │
│     └─► Write validation_*.spec                                 │
│     └─► RUN: certoraRun validation config                       │
│     └─► ALL MUST PASS before continuing                         │
│                                                                  │
│  6. SANITY GATE (Phase 6)                                       │
│     └─► Complete checklist in spec_authoring.md                 │
│                                                                  │
│  7. WRITE CVL (Phase 7)                                         │
│     └─► NOW write {Contract}.spec                               │
│     └─► RUN: certoraRun real config                             │
│                                                                  │
│  8. DEBUG (if needed)                                           │
│     └─► Use CE_DIAGNOSIS_FRAMEWORK.md                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## EXAMPLE: Applying to a New Contract

Let's say you want to verify `EmergencyProtectedTimelock.sol`:

```bash
# Step 1: Create files
mkdir -p spec_authoring
touch spec_authoring/timelock_spec_authoring.md
touch spec_authoring/timelock_candidate_properties.md
touch spec_authoring/timelock_causal_validation.md
touch certora/specs/validation_timelock.spec
touch certora/confs/validation_timelock.conf

# Step 2: Analyze (fill spec_authoring.md)
grep -n "function.*external" contracts/EmergencyProtectedTimelock.sol

# Step 3: Properties (fill candidate_properties.md)
# - "Proposals can only be executed after delay"
# - "Only governance can schedule"
# etc.

# Step 4: Classify each as INVARIANT or RULE

# Step 5: Write validation rules and run
certoraRun certora/confs/validation_timelock.conf

# Step 6: Complete sanity checklist

# Step 7: Write real spec
touch certora/specs/EmergencyProtectedTimelock.spec
certoraRun certora/confs/EmergencyProtectedTimelock.conf
```

---

## FILES IN YOUR FRAMEWORK

```
your-project/
├── certora-quickstart-template.md    ← THIS FILE (how to apply)
├── certora-master-guide.md           ← Complete step-by-step instructions
├── cvl-language-deep-dive.md         ← CVL language reference ⭐ NEW v1.5
├── verification-playbooks.md         ← Worked examples ⭐ NEW v1.5
├── certora-workflow.md               ← Step-by-step process
├── certora-spec-framework.md         ← CVL templates & patterns
├── certora-ce-diagnosis-framework.md ← Debugging counterexamples
├── SPEC AUTHORING (CERTORA).md       ← Deep methodology
├── categorizing-properties.md        ← Property discovery
├── best-practices-from-certora.md    ← Proven techniques
├── advanced-cli-reference.md         ← CLI & performance ⭐ NEW v1.4
├── quick-reference-v1.3.md           ← Printable cheat sheet
├── index.md                          ← Navigation guide
├── version-history.md                ← Version tracking
│
├── spec_authoring/
│   ├── {contract}_spec_authoring.md
│   ├── {contract}_candidate_properties.md
│   └── {contract}_causal_validation.md
│
└── certora/
    ├── specs/
    │   ├── validation_{contract}.spec
    │   └── {Contract}.spec
    ├── confs/
    │   ├── validation_{contract}.conf
    │   └── {Contract}.conf
    ├── harnesses/
    └── helpers/
```
