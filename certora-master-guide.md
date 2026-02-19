# CERTORA VERIFICATION MASTER GUIDE

> **The Complete Framework for Formal Verification of Smart Contracts**  
> **Version:** 2.0 ( Red Team Hardening + Validation Evidence Gate)  
> **Use this guide to verify ANY Solidity contract from scratch**

---

## TABLE OF CONTENTS

1. [Framework Overview](#1-framework-overview)
2. [Project Setup](#2-project-setup)
3. [Phase 0: Contract Analysis](#3-phase-0-contract-analysis)
4. [Phase -1: Execution Closure](#4-phase--1-execution-closure)
5. [Phase 2: Property Discovery](#5-phase-2-property-discovery)
6. [Phase 2.5: Classification](#6-phase-25-classification)
7. [Phase 3.5: Causal Validation](#7-phase-35-causal-validation)
8. [Phase 4-6: Modeling & Sanity](#8-phase-4-6-modeling--sanity)
9. [Phase 7: Write CVL](#9-phase-7-write-cvl)
10. [Running & Debugging](#10-running--debugging)
11. [Templates](#11-templates)
12. [Quick Reference](#12-quick-reference)
13. [Quick Start Chat Prompt](#13-quick-start-chat-prompt)

---

# 1. FRAMEWORK OVERVIEW

## 1.1 Your Framework Documents

| Document | Purpose | When to Use |
|----------|---------|-------------|
| **certora-master-guide.md** | Complete step-by-step instructions | Starting any new verification |
| **cvl-language-deep-dive.md** | Complete CVL language reference | ← **NEW in v1.5** |
| **verification-playbooks.md** | Worked examples (ERC-20, WETH, ERC-721) | ← **NEW in v1.5** |
| **advanced-cli-reference.md** | Performance optimization & advanced flags | ← **NEW in v1.4** |
| **SPEC AUTHORING (CERTORA).md** | Deep methodology & theory | Understanding WHY |
| **categorizing-properties.md** | Property discovery guidance | Phase 2 |
| **certora-spec-framework.md** | CVL 2.0 syntax & templates | Writing actual CVL |
| **certora-ce-diagnosis-framework.md** | Counterexample debugging | When rules fail |
| **certora-workflow.md** | Phase overview | Quick reference |
| **best-practices-from-certora.md** | Official tutorial techniques | ← **NEW in v1.3** |

## 1.2 The Golden Rule

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   UNDERSTAND → ENUMERATE → VALIDATE → WRITE → DEBUG              │
│                                                                  │
│   ❌ Never write CVL before completing phases 0 through 3.5     │
│   ❌ Never skip causal validation                                │
│   ❌ Never assume external contracts are "standard"              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 1.3 Workflow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CERTORA WORKFLOW                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │
│  │  PHASE 0    │───▶│  PHASE -1   │───▶│  PHASE 2    │               │
│  │  Contract   │    │  Execution  │    │  Property   │               │
│  │  Analysis   │    │  Closure    │    │  Discovery  │               │
│  └─────────────┘    └─────────────┘    └─────────────┘               │
│         │                  │                  │                       │
│         ▼                  ▼                  ▼                       │
│  ┌─────────────────────────────────────────────────┐                 │
│  │           {contract}_spec_authoring.md          │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│                            │                                          │
│                            ▼                                          │
│                   ┌─────────────┐                                     │
│                   │  PHASE 2.5  │                                     │
│                   │  INVARIANT  │                                     │
│                   │  vs RULE    │                                     │
│                   └─────────────┘                                     │
│                            │                                          │
│                            ▼                                          │
│  ┌─────────────────────────────────────────────────┐                 │
│  │         {contract}_candidate_properties.md      │                 │
│  └─────────────────────────────────────────────────┘                 │
│                                                                       │
│                            │                                          │
│                            ▼                                          │
│                   ┌─────────────┐                                     │
│                   │  PHASE 3.5  │◀──── RUN VALIDATION                │
│                   │   Causal    │      certoraRun validation.conf    │
│                   │  Validation │                                     │
│                   └─────────────┘                                     │
│                            │                                          │
│                    PASS?   │                                          │
│               ┌────────────┴────────────┐                            │
│               │ NO                      │ YES                        │
│               ▼                         ▼                            │
│        ┌──────────┐            ┌─────────────┐                       │
│        │  FIX     │            │  PHASE 4-6  │                       │
│        │  MODELING│            │  Modeling & │                       │
│        │  GAP     │            │  Sanity     │                       │
│        └──────────┘            └─────────────┘                       │
│               │                         │                            │
│               └─────────────────────────┤                            │
│                                         ▼                            │
│                                ┌─────────────┐                       │
│                                │  PHASE 7    │                       │
│                                │  Write CVL  │                       │
│                                └─────────────┘                       │
│                                         │                            │
│                                         ▼                            │
│                                    RUN PROVER                        │
│                                certoraRun {Contract}.conf            │
│                                         │                            │
│                              ┌──────────┴──────────┐                 │
│                              │                     │                 │
│                         ❌ FAIL              ✅ PASS                 │
│                              │                     │                 │
│                              ▼                     ▼                 │
│                    ┌──────────────┐         ┌──────────┐            │
│                    │ CE DIAGNOSIS │         │  DONE!   │            │
│                    │ FRAMEWORK    │         │          │            │
│                    └──────────────┘         └──────────┘            │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

# 2. PROJECT SETUP

## 2.0 Understanding What "New Contract" Means

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        KEY TERMINOLOGY                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PRIMARY TARGET     = The contract you write invariants/rules FOR        │
│                       (e.g., Main.sol, Vault.sol, Escrow.sol)           │
│                                                                          │
│  COMPILATION DEPS   = ALL files needed to compile the target             │
│                       (imports, inherited contracts, libraries)          │
│                                                                          │
│  IN-SCOPE CONTRACTS = Contracts whose behavior you model accurately      │
│                       (determined during Phase -1)                       │
│                                                                          │
│  EXTERNAL CONTRACTS = Contracts outside your codebase                    │
│                       (ERC20 tokens, oracles, protocols)                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

IMPORTANT: You need the ENTIRE contracts folder for compilation,
           but you VERIFY one primary target at a time.
```

## 2.1 Generic Folder Structure Template

**Copy this structure for ANY new formal verification project:**

```
{PROJECT_NAME}/                        ← Your verification project root
│
├── ══════════════════════════════════════════════════════════════════════
│   FRAMEWORK FILES (Copy these to every new project)
├── ══════════════════════════════════════════════════════════════════════
│
├── certora-master-guide.md            ← This file - START HERE
├── certora-workflow.md                ← Phase overview
├── certora-spec-framework.md          ← CVL 2.0 syntax & templates
├── certora-ce-diagnosis-framework.md  ← Counterexample debugging
├── SPEC AUTHORING (CERTORA).md        ← Deep methodology
├── categorizing-properties.md         ← Phase 2 property discovery
│
├── Certora-CVL-Documentation/                           ← Reference documentation
│   └── ...
│
├── ══════════════════════════════════════════════════════════════════════
│   ORIGINAL CONTRACTS (The protocol's source code)
├── ══════════════════════════════════════════════════════════════════════
│
├── contracts/                         ← OR whatever the project uses
│   ├── {path}/{to}/{target}/
│   │   └── {TargetContract}.sol       ← PRIMARY TARGET
│   ├── {other}/
│   │   └── {dependencies}.sol         ← Compilation dependencies
│   └── {interfaces}/
│       └── {interfaces}.sol           ← Interface files
│
├── ══════════════════════════════════════════════════════════════════════
│   SPEC AUTHORING WORKSPACE (Your analysis - one per target)
├── ══════════════════════════════════════════════════════════════════════
│
├── spec_authoring/
│   │
│   │   # For each PRIMARY TARGET, create these 3 files:
│   │
│   ├── {target}_spec_authoring.md      ← Phases 0, -1, 3, 4, 5, 6
│   ├── {target}_candidate_properties.md ← Phase 2, 2.5
│   └── {target}_causal_validation.md   ← Phase 3.5
│
├── ══════════════════════════════════════════════════════════════════════
│   CERTORA VERIFICATION (CVL specs and configs)
├── ══════════════════════════════════════════════════════════════════════
│
└── certora/
    │
    ├── specs/
    │   ├── validation_{target}.spec   ← Causal validation (run FIRST)
    │   └── {TargetContract}.spec      ← Real verification spec
    │
    ├── confs/
    │   ├── validation_{target}.conf   ← Config for validation
    │   └── {TargetContract}.conf      ← Config for real verification
    │
    ├── harnesses/                     ← Simplified external contracts
    │   └── DummyERC20.sol
    │
    └── helpers/                       ← Helper contracts for verification
        └── ...
```

## 2.2 Example: Real Project Structure

**Example: Verifying `Main.sol` in a DEX protocol**

```
dex-verification/
│
├── ═══ FRAMEWORK FILES ═══════════════════════════════════════════════════
├── certora-master-guide.md
├── certora-workflow.md
├── certora-spec-framework.md
├── certora-ce-diagnosis-framework.md
├── SPEC AUTHORING (CERTORA).md
├── categorizing-properties.md
│
├── ═══ ORIGINAL CONTRACTS ════════════════════════════════════════════════
├── protocols/
│   └── dexV2/
│       ├── base/
│       │   ├── core/
│       │   │   ├── adminModule.sol     ← Compilation dependency
│       │   │   ├── helpers.sol         ← Compilation dependency
│       │   │   └── main.sol            ← ⭐ PRIMARY TARGET
│       │   └── other/
│       │       ├── commonImport.sol    ← Compilation dependency
│       │       ├── error.sol           ← Compilation dependency
│       │       ├── errorTypes.sol      ← Compilation dependency
│       │       ├── events.sol          ← Compilation dependency
│       │       ├── interfaces.sol      ← Compilation dependency
│       │       └── variables.sol       ← Compilation dependency
│       └── proxy.sol                   ← May need modeling
│
├── ═══ SPEC AUTHORING (Analysis for main.sol) ════════════════════════════
├── spec_authoring/
│   ├── main_spec_authoring.md          ← Analysis document
│   ├── main_candidate_properties.md    ← Properties list
│   └── main_causal_validation.md       ← Mutation path analysis
│
├── ═══ CERTORA VERIFICATION ══════════════════════════════════════════════
└── certora/
    ├── specs/
    │   ├── validation_main.spec        ← Run FIRST
    │   └── Main.spec                   ← Real verification
    ├── confs/
    │   ├── validation_main.conf
    │   └── Main.conf
    └── harnesses/
        └── DummyERC20.sol              ← If Main interacts with tokens
```

## 2.3 Setup Commands

```bash
# ═══════════════════════════════════════════════════════════════════════════
# STEP 1: Define your variables (CHANGE THESE)
# ═══════════════════════════════════════════════════════════════════════════

# The contract name (as it appears in Solidity: contract XYZ)
TARGET_CONTRACT="Main"

# Lowercase version for filenames
TARGET_LOWER="main"

# Your project root (where contracts already exist)
PROJECT_ROOT="/path/to/your/project"

# ═══════════════════════════════════════════════════════════════════════════
# STEP 2: Create verification directories
# ═══════════════════════════════════════════════════════════════════════════

cd "$PROJECT_ROOT"

# Create spec authoring workspace
mkdir -p spec_authoring

# Create certora directories
mkdir -p certora/specs
mkdir -p certora/confs
mkdir -p certora/harnesses
mkdir -p certora/helpers

# ═══════════════════════════════════════════════════════════════════════════
# STEP 3: Create analysis documents for your target
# ═══════════════════════════════════════════════════════════════════════════

touch "spec_authoring/${TARGET_LOWER}_spec_authoring.md"
touch "spec_authoring/${TARGET_LOWER}_candidate_properties.md"
touch "spec_authoring/${TARGET_LOWER}_causal_validation.md"

# ═══════════════════════════════════════════════════════════════════════════
# STEP 4: Create CVL spec and conf files
# ═══════════════════════════════════════════════════════════════════════════

touch "certora/specs/validation_${TARGET_LOWER}.spec"
touch "certora/specs/${TARGET_CONTRACT}.spec"
touch "certora/confs/validation_${TARGET_LOWER}.conf"
touch "certora/confs/${TARGET_CONTRACT}.conf"

# ═══════════════════════════════════════════════════════════════════════════
# STEP 5: Copy framework files (if not already present)
# ═══════════════════════════════════════════════════════════════════════════

# Copy from your template location (adjust path as needed)
# cp /path/to/templates/CERTORA_*.md .
# cp /path/to/templates/categorizing-properties.md .
# cp /path/to/templates/"SPEC AUTHORING (CERTORA).md" .

echo "✅ Verification structure created for ${TARGET_CONTRACT}"
echo ""
echo "Next steps:"
echo "1. Open spec_authoring/${TARGET_LOWER}_spec_authoring.md"
echo "2. Follow certora-master-guide.md Phase 0"
```

## 2.4 Configuration File Template

**Template for `certora/confs/{TargetContract}.conf`:**

```json
{
    "files": [
        "path/to/target/TargetContract.sol",
        "path/to/dependency1.sol",
        "path/to/dependency2.sol",
        "path/to/interfaces/IContract.sol",
        "certora/harnesses/DummyERC20.sol"
    ],
    
    "link": [
        "TargetContract:TOKEN=DummyERC20"
    ],
    
    "verify": "TargetContract:certora/specs/TargetContract.spec",
    
    "msg": "TargetContract Verification",
    
    "packages": [
        "@openzeppelin=lib/openzeppelin-contracts",
        "@openzeppelin=node_modules/@openzeppelin"
    ],
    
    "solc_evm_version": "cancun",
    "solc": "solc",
    "optimistic_loop": true,
    "optimistic_fallback": true,
    "loop_iter": "3",
    "rule_sanity": "basic",
    "build_cache": true,
    "server": "production"
}
```

**Key points:**
- `files`: List ALL files needed to compile the target (target + all dependencies)
- `verify`: Specifies which contract is the PRIMARY TARGET
- `link`: Connect contract references to harnesses

## 2.5 How Compilation Dependencies Work

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Q: How does Certora know which files to include?                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│ A: YOU list them in the "files" array. Certora does NOT auto-resolve.   │
│                                                                          │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ main.sol                                                             │ │
│ │   import "./helpers.sol";        ← Must be in files[]               │ │
│ │   import "../other/variables.sol"; ← Must be in files[]             │ │
│ │   import "../other/interfaces.sol"; ← Must be in files[]            │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│ If you miss a file, you'll get compilation errors.                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2.6 Verifying Multiple Contracts

If you need to verify multiple targets in the same project:

```
project/
├── spec_authoring/
│   │
│   │   # Target 1: Main.sol
│   ├── main_spec_authoring.md
│   ├── main_candidate_properties.md
│   ├── main_causal_validation.md
│   │
│   │   # Target 2: AdminModule.sol
│   ├── adminmodule_spec_authoring.md
│   ├── adminmodule_candidate_properties.md
│   └── adminmodule_causal_validation.md
│
└── certora/
    ├── specs/
    │   ├── validation_main.spec
    │   ├── Main.spec
    │   ├── validation_adminmodule.spec
    │   └── AdminModule.spec
    │
    └── confs/
        ├── validation_main.conf
        ├── Main.conf
        ├── validation_adminmodule.conf
        └── AdminModule.conf
```

**Each target gets its own:**
- 3 spec_authoring documents
- 2 spec files (validation + real)
- 2 conf files (validation + real)

---

# 3. PHASE 0: CONTRACT ANALYSIS

> **Goal:** Extract execution reality from the contract

## 3.1 Commands to Gather Information

```bash
# ═══════════════════════════════════════════════════════════════
# Find all external/public functions (entry points)
# ═══════════════════════════════════════════════════════════════
grep -n "function.*external\|function.*public" contracts/$CONTRACT_FILE

# ═══════════════════════════════════════════════════════════════
# Find all storage variables
# ═══════════════════════════════════════════════════════════════
grep -n "^\s*mapping\|^\s*uint\|^\s*int\|^\s*address\|^\s*bool\|^\s*bytes\|^\s*string" contracts/$CONTRACT_FILE

# ═══════════════════════════════════════════════════════════════
# Find all external calls (potential trust boundaries)
# ═══════════════════════════════════════════════════════════════
grep -n "\.[a-zA-Z]*(" contracts/$CONTRACT_FILE | grep -v "//" | grep -v "this\."

# ═══════════════════════════════════════════════════════════════
# Find all imports (execution universe)
# ═══════════════════════════════════════════════════════════════
grep -n "^import" contracts/$CONTRACT_FILE

# ═══════════════════════════════════════════════════════════════
# Find all events (state change indicators)
# ═══════════════════════════════════════════════════════════════
grep -n "emit " contracts/$CONTRACT_FILE

# ═══════════════════════════════════════════════════════════════
# Find all modifiers (access control)
# ═══════════════════════════════════════════════════════════════
grep -n "modifier\|onlyOwner\|onlyAdmin\|require.*msg.sender" contracts/$CONTRACT_FILE
```

## 3.2 Fill in `{contract}_spec_authoring.md`

Copy this template and fill in the blanks:

```markdown
# [CONTRACT_NAME] Specification Authoring Workspace

> **Contract:** `[ContractName].sol`
> **Date Started:** [DATE]
> **Author:** [YOUR NAME]

---

## PHASE 0: VERIFICATION SCOPE

### 0.1 Verification Boundary

**Primary Contract:** `[ContractName].sol`

**In-Scope Contracts:**
| Contract | Role | Why In-Scope |
|----------|------|--------------|
| [ContractName] | Primary | Main verification target |
| [Contract2] | [Role] | [Reason] |

**Out-of-Scope Contracts:**
| Contract | Why Out-of-Scope | Modeling Required |
|----------|------------------|-------------------|
| [Contract] | [Reason] | [DISPATCHER/NONDET/HAVOC] |

---

### 0.2 Entry Points (State-Changing Functions)

| # | Function | Visibility | Modifiers | State Changes | External Calls |
|---|----------|------------|-----------|---------------|----------------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |

**Total Entry Points:** [N]

---

### 0.3 View Functions

| Function | Returns | Security-Critical? | Used in require()? |
|----------|---------|-------------------|-------------------|
| | | Yes/No | Yes/No |

> ⚠️ View functions used in require() are security-critical entry points

---

### 0.4 State Mutation Map

| Storage Variable | Type | Modified By | Read By |
|------------------|------|-------------|---------|
| | | | |

---

### 0.5 Asset Flow Trace

For each asset type:

**Asset: [ETH / ERC20 / ERC721 / etc.]**
| Aspect | Value |
|--------|-------|
| Owning Contract | |
| Inflow Functions | |
| Outflow Functions | |
| Balance Check Functions | |
| Reentrancy Risk? | Yes/No |

---

## PHASE -1: EXECUTION CLOSURE

### -1.1 Execution Universe

**All contracts that participate in execution:**

| Contract | Interaction Type | Called By | Calls To |
|----------|------------------|-----------|----------|
| | Direct Call | | |
| | Callback | | |
| | Delegate | | |

---

### -1.2 Interaction Ownership Table

| External Contract | Owns What Truth | Our Contract Reads | Our Contract Writes | Callbacks? |
|-------------------|-----------------|-------------------|--------------------| -----------|
| | | | | |

> 🚨 Every row must be complete. Blank rows invalidate the spec.

---

### -1.3 Modeling Obligations

| Truth | Owner | Modeling Decision | Justification |
|-------|-------|-------------------|---------------|
| balances | ERC20 | DISPATCHER | Need accurate balance tracking |
| prices | Oracle | NONDET / TRUSTED | [Justify choice] |
| | | | |

---

## PHASE 3: STATE CLASSIFICATION

### Trusted State (Owned by this contract)
- [ ] [variable1]
- [ ] [variable2]

### Untrusted State (External)
- [ ] [external.variable1] - Owner: [Contract]
- [ ] [external.variable2] - Owner: [Contract]

---

## PHASE 4: MODELING DECISIONS

### DISPATCHER Summaries
| Contract.Function | Routes To | Why DISPATCHER |
|-------------------|-----------|----------------|
| | | |

### NONDET Summaries
| Contract.Function | Why NONDET is Safe |
|-------------------|-------------------|
| | Does not affect invariant because... |

### Explicit Constraints
| Constraint | Why Needed |
|------------|------------|
| | |

---

## PHASE 5: GHOST REQUIREMENTS

### Ghosts Needed
| Ghost Name | Type | Purpose | Hooks On |
|------------|------|---------|----------|
| | mathint | Sum of... | Sstore X |

### Ghosts NOT Needed (Documented)
| Considered | Why Not Needed |
|------------|----------------|
| | Can read directly from storage |

---

## PHASE 6: SANITY GATE

### Execution Closure ✓
- [ ] All entry points enumerated
- [ ] All state mutations mapped
- [ ] All external reads modeled
- [ ] All external writes modeled
- [ ] NONDET usages justified

### Causal Closure ✓
- [ ] All mutation paths for invariant variables identified
- [ ] All ghosts have complete hooks
- [ ] Constructor effects modeled
- [ ] Validation rules written and PASSED

### Bounded State ✓
- [ ] Array lengths bounded (< 100 or realistic)
- [ ] Timestamps bounded (<= max_uint40)
- [ ] Counters bounded realistically

### Property Quality ✓
- [ ] No property mixes two truth owners
- [ ] No ghost mirrors readable storage
- [ ] No hidden trust assumptions
- [ ] Every property has real exploit if broken

---

## NOTES & DECISIONS LOG

| Date | Decision | Rationale |
|------|----------|-----------|
| | | |
```

---

# 4. PHASE -1: EXECUTION CLOSURE

> **Goal:** Ensure every external interaction is modeled

## 4.1 Key Questions

For EACH external contract called:

1. **What truth does it own?** (balances, prices, ownership, etc.)
2. **Do we read from it?** (view calls, storage reads)
3. **Do we write to it?** (state changes, transfers)
4. **Can it call us back?** (reentrancy, callbacks)
5. **How should we model it?** (DISPATCHER, NONDET, HAVOC)

## 4.2 Modeling Decision Matrix

| External Contract Type | Default Modeling | When to Change |
|------------------------|------------------|----------------|
| ERC20 Token | DISPATCHER | Never use NONDET for balances |
| ERC721 NFT | DISPATCHER | Need ownership tracking |
| Oracle | NONDET or TRUSTED | Document trust assumption |
| Lending Protocol | DISPATCHER | Never assume solvency |
| Governance | DISPATCHER | Need state tracking |
| Unknown Contract | HAVOC_ALL | Maximum adversarial |

## 4.3 Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Not modeling token | Balance can be anything | Add DISPATCHER for token |
| NONDET for balances | Spurious counterexamples | Use DISPATCHER |
| Missing callback | Reentrancy not caught | Model callback pattern |
| Assuming oracle honest | Miss price manipulation | Document trust or use NONDET |

---

# 5. PHASE 2: PROPERTY DISCOVERY

> **Goal:** List all security properties in plain English

## 5.1 Use categorizing-properties.md + Best Practices

Follow the template in `categorizing-properties.md` to discover properties.

**NEW in v1.3:** Also reference `best-practices-from-certora.md` Section 1 for:
- Property discovery techniques
- The 4 fatal mistakes to avoid
- Anti-pattern: mimicking implementation
- Iterative discovery process

**Property Prioritization (v1.3):**
After discovering properties, assign priority levels using `categorizing-properties.md` Section 7:
- **HIGH**: Loss of funds, DoS, privilege escalation
- **MEDIUM**: Accounting integrity, solvency
- **LOW**: Single function correctness

Prioritization guides verification order and time allocation.

## 5.2 Fill in `{contract}_candidate_properties.md`

```markdown
# [CONTRACT_NAME] Candidate Security Properties

> **Contract:** `[ContractName].sol`
> **Date:** [DATE]
> **Phase:** 2 (Property Discovery)

---

## Category A: Asset Safety / Solvency

### A1. [Property Name]
**Plain English:** [One sentence]
**Impact if Violated:** [Theft / Insolvency / Loss of Funds]
**Category:** Valid State
**Priority:** [HIGH / MEDIUM / LOW]  ← NEW v1.3
**Variables Involved:**
  - `[variable]` (owned by [contract])
**External Truths Needed:** [None / List them]
**Aggregate/History Required?:** [Yes/No]

### A2. ...

---

## Category B: Functional Correctness

### B1. [Property Name]
**Plain English:** When [trigger], [expected outcome]
**Impact if Violated:** [Incorrect behavior / User harm]
**Category:** State Transition
**Variables Involved:**
  - `[variable]` (owned by [contract])
**External Truths Needed:** [None / List them]
**Aggregate/History Required?:** [No - function-specific]

---

## Category C: State Consistency

### C1. [Property Name]
**Plain English:** [Relationship that must always hold]
**Impact if Violated:** [Accounting corruption / Inconsistency]
**Category:** System-Level
**Variables Involved:**
  - `[variable1]` (owned by [contract])
  - `[variable2]` (owned by [contract])
**External Truths Needed:** [None / List them]
**Aggregate/History Required?:** [Yes - needs ghost for sum]

---

## Category D: Access Control

### D1. [Property Name]
**Plain English:** Only [role] can [action]
**Impact if Violated:** [Privilege escalation / Unauthorized access]
**Category:** State Transition
**Variables Involved:**
  - `[protected variable]`
**External Truths Needed:** [None / List them]
**Aggregate/History Required?:** [No]

---

## Category E: State Machine

### E1. [Property Name]
**Plain English:** State can only transition [from] → [to]
**Impact if Violated:** [Invalid state / Protocol corruption]
**Category:** State Transition
**Variables Involved:**
  - `state` variable
**External Truths Needed:** [None]
**Aggregate/History Required?:** [No]

---

## Out of Scope (Trusted Role Assumptions)

### X1. [Property Name]
**Plain English:** [What would be violated]
**Why Out of Scope:** Requires [trusted role] to act maliciously
**Trust Assumption:** [Role] is honest

---

## Summary

| ID | Name | Category | Priority | Type (TBD) | Ghost Needed? |
|----|------|----------|----------|------------|---------------|
| A1 | | Valid State | HIGH | ? | No |
| A2 | | Valid State | MEDIUM | ? | No |
| B1 | | Transition | HIGH | ? | No |
| C1 | | System-Level | MEDIUM | ? | Yes |
| D1 | | Transition | HIGH | ? | No |
| E1 | | Transition | LOW | ? | No |
```

---

# 6. PHASE 2.5: CLASSIFICATION

> **Goal:** Decide INVARIANT vs RULE for each property

## 6.1 Decision Flowchart

For EACH property, follow this flowchart:

```
┌─────────────────────────────────────────────────────────────┐
│ Q0: Is all state in this property owned or modeled?         │
├─────────────────────────────────────────────────────────────┤
│ NO  → STOP. Model the external state first.                 │
│ YES → Continue                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Q1: Is the question "Can X EVER happen?"                    │
│     vs "WHEN Y happens, does Z happen?"                     │
├─────────────────────────────────────────────────────────────┤
│ EVER  → INVARIANT                                           │
│ WHEN  → Continue                                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Q2: Does it depend on a specific function call?             │
├─────────────────────────────────────────────────────────────┤
│ YES → RULE                                                  │
│ NO  → Continue                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Q2.5: Does it depend on external state not fully modeled?   │
│       (ERC20 balances, oracle prices, etc.)                 │
├─────────────────────────────────────────────────────────────┤
│ YES → RULE (invariants over external state are dangerous)   │
│ NO  → Continue                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Q3: Would a single violation break the protocol forever?    │
├─────────────────────────────────────────────────────────────┤
│ YES → INVARIANT                                             │
│ NO  → RULE                                                  │
└─────────────────────────────────────────────────────────────┘

📌 When in doubt, default to RULE. It's safer.
```

## 6.2 Update Your Properties

Add the classification to each property:

```markdown
### A1. ETH Solvency
**Plain English:** Contract always has enough ETH to pay users
...
**Classification:** INVARIANT ✓
**Reasoning:** "Can contract EVER owe more than it has?" = EVER question
```

---

# 7. PHASE 3.5: CAUSAL VALIDATION

> **Goal:** Prove mutation paths are complete BEFORE writing real spec

## 7.1 Fill in `{contract}_causal_validation.md`

```markdown
# [CONTRACT_NAME] Causal Validation

> **Purpose:** Enumerate ALL mutation paths for INVARIANT variables
> **Contract:** `[ContractName].sol`
> **Date:** [DATE]

---

## Property: [PROPERTY_NAME]

**Type:** INVARIANT
**From:** [candidate_properties.md reference]

### Variables in This Property

| Variable | Type | Location | Purpose |
|----------|------|----------|---------|
| `var1` | uint256 | Storage | [Purpose] |
| `var2` | mathint | Ghost | [Purpose] |

### Mutation Paths for `var1`

| # | Function | Effect | Direction |
|---|----------|--------|-----------|
| 1 | `function1()` | [what happens] | increase |
| 2 | `function2()` | [what happens] | decrease |
| 3 | `constructor()` | [initial value] | initialize |

### Mutation Paths for `var2` (Ghost)

| # | Storage Write | Hook Location | Effect |
|---|---------------|---------------|--------|
| 1 | `_balances[user]` | Sstore hook | sum += new - old |
| 2 | constructor | init_state axiom | = 0 |

### Validation Rule

```cvl
rule validation_mutation_paths_var1(method f)
    filtered { f -> f.contract == currentContract && !f.isView }
{
    uint256 before = var1;
    
    env e;
    calldataarg args;
    f(e, args);
    
    uint256 after = var1;
    
    assert before != after => (
        f.selector == sig:function1().selector ||
        f.selector == sig:function2().selector
    ), "Unmodeled mutation path for var1";
}
```

### Checklist

- [ ] **Reachability: `satisfy` rules written for every state-changing function**  
- [ ] **Reachability: ALL `satisfy` rules PASS (no always-reverting functions)**  
- [ ] **Failure-path reachability: `satisfy lastReverted` rules for critical revert conditions** 
- [ ] **Failure-path reachability: ALL failure-path `satisfy` rules PASS** 
- [ ] All mutation paths enumerated
- [ ] All paths have hooks (if ghost)
- [ ] Constructor modeled
- [ ] Validation rule written
- [ ] Validation rule PASSES ✓
- [ ] **Evidence Review: satisfy witnesses inspected — non-degenerate**
- [ ] **Evidence Review: ghost sync witnesses inspected — non-trivial** 
- [ ] **Evidence Review: mutation whitelists match Phase 0 entry points** 
- [ ] **Evidence Review: advanced sanity run (rule_sanity: advanced) passed** 
- [ ] **Evidence Review: sign-off completed in causal_validation.md** 
```

## 7.2 Create `validation_{contract}.spec`

```cvl
/*
 * ═══════════════════════════════════════════════════════════════
 * [CONTRACT_NAME] CAUSAL VALIDATION SPEC
 * ═══════════════════════════════════════════════════════════════
 * 
 * Purpose: Validate causal closure BEFORE writing real spec
 * 
 * WORKFLOW:
 * 1. Run: certoraRun certora/confs/validation_[contract].conf
 * 2. ALL rules must PASS
 * 3. If any FAIL → Fix modeling gap, re-run
 * 4. Only after ALL PASS → Complete Validation Evidence Review (Section 7.5)
 * 5. Only after Evidence Review signed off → Proceed to real spec
 * ═══════════════════════════════════════════════════════════════
 */

using YourContract as currentContract;

// ═══════════════════════════════════════════════════════════════
// METHODS BLOCK
// ═══════════════════════════════════════════════════════════════

methods {
    // Contract functions (list all state-changing)
    function function1() external;
    function function2(uint256) external;
    function function3(address, uint256) external;
    
    // View functions (envfree where possible)
    function viewFunction() external returns (uint256) envfree;
    
    // External contracts - DISPATCHER
    function _.transfer(address, uint256) external => DISPATCHER(true);
    function _.balanceOf(address) external => DISPATCHER(true);
    
    // Unrelated external calls - NONDET (justified)
    function _.unrelatedCall() external => NONDET;
}

// ═══════════════════════════════════════════════════════════════
// GHOSTS (if needed for validation)
// ═══════════════════════════════════════════════════════════════

ghost mathint sumVariable {
    init_state axiom sumVariable == 0;
}

// ═══════════════════════════════════════════════════════════════
// HOOKS
// ═══════════════════════════════════════════════════════════════

hook Sstore currentContract.mapping[KEY address user] uint256 newVal (uint256 oldVal) {
    sumVariable = sumVariable + newVal - oldVal;
}

// ═══════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ═══════════════════════════════════════════════════════════════

function getVariable() returns uint256 {
    return currentContract.variable;
}

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULE 0: Function Reachability (satisfy)        NEW v1.8
// ═══════════════════════════════════════════════════════════════
// Before proving ANY assert, prove each function CAN execute
// without reverting. If satisfy fails → function always reverts
// → every assert rule is vacuously true (proves nothing).
//
// ⚠️ IMPORTANT: satisfy !lastReverted proves LIVENESS (a non-
// reverting path exists), NOT EFFECT (the function changes state).
// Effect is validated by Validation Rule 1 (mutation path rules).
// BOTH are required — do not treat satisfy alone as proof of
// functional correctness.
// ═══════════════════════════════════════════════════════════════

rule validation_reachability_function1() {
    env e;
    // Add realistic caller constraints
    require e.msg.sender != 0;
    require e.msg.value == 0;  // if non-payable

    function1@withrevert(e);
    satisfy !lastReverted, "function1 is reachable (non-reverting path exists)";
}

rule validation_reachability_function2(uint256 amount) {
    env e;
    require e.msg.sender != 0;
    require e.msg.value == 0;

    function2@withrevert(e, amount);
    satisfy !lastReverted, "function2 is reachable (non-reverting path exists)";
}

// Repeat for EVERY state-changing function.
// If ANY satisfy rule is VIOLATED → STOP.
// The function always reverts under your modeling,
// which means every assert rule for it is vacuous.
// Fix: check harness, DISPATCHER config, or require statements.

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULE 0b: Failure-Path Reachability (satisfy)   NEW v1.9
// ═══════════════════════════════════════════════════════════════
// For each critical revert condition, prove the revert path is
// reachable. Without this, a biconditional `<=>` in your real
// spec may pass vacuously because the revert scenario is
// impossible under your modeling.
// ═══════════════════════════════════════════════════════════════

rule validation_revert_reachability_function1_unauthorized() {
    env e;
    require e.msg.sender != authorizedAddress();
    require e.msg.value == 0;

    function1@withrevert(e);
    satisfy lastReverted, "function1 reverts for unauthorized caller (revert path is reachable)";
}

rule validation_revert_reachability_function2_zero_amount() {
    env e;
    require e.msg.sender != 0;
    require e.msg.value == 0;

    function2@withrevert(e, 0);
    satisfy lastReverted, "function2 reverts for zero amount (revert path is reachable)";
}

// Repeat for EVERY critical revert condition documented in Phase 3.
// If ANY satisfy-revert rule is VIOLATED → the revert path is
// unreachable under your modeling, and your biconditional `<=>`
// revert rules will pass vacuously. Fix BEFORE writing real spec.

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULE 1: Mutation Paths for [Variable]
// ═══════════════════════════════════════════════════════════════

rule validation_mutation_paths_variable(method f)
    filtered { f -> f.contract == currentContract && !f.isView }
{
    uint256 before = getVariable();
    
    env e;
    calldataarg args;
    f(e, args);
    
    uint256 after = getVariable();
    
    assert before != after => (
        f.selector == sig:function1().selector ||
        f.selector == sig:function2(uint256).selector
    ), "CAUSAL VIOLATION: Unmodeled mutation path";
}

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULE 2: Ghost Synchronization
// ═══════════════════════════════════════════════════════════════

rule validation_ghost_sync(method f)
    filtered { f -> f.contract == currentContract }
{
    // Require sync before
    require to_mathint(getTotalFromStorage()) == sumVariable;
    
    env e;
    calldataarg args;
    f(e, args);
    
    // Must remain synced
    assert to_mathint(getTotalFromStorage()) == sumVariable,
        "GHOST DESYNC: Hook missing or incorrect";
}

// ═══════════════════════════════════════════════════════════════
// VALIDATION RULE 3: State Bounds
// ═══════════════════════════════════════════════════════════════

invariant validation_stateBounds()
    getState() <= MAX_STATE_VALUE
```

## 7.3 Create `validation_{contract}.conf`

```json
{
    "files": [
        "contracts/YourContract.sol",
        "contracts/Dependency.sol",
        "certora/harnesses/DummyToken.sol"
    ],
    "link": [
        "YourContract:TOKEN=DummyToken",
        "YourContract:DEPENDENCY=Dependency"
    ],
    "msg": "[ContractName] Causal Validation",
    "packages": [
        "@openzeppelin=lib/openzeppelin-contracts",
        "@openzeppelin=node_modules/@openzeppelin"
    ],
    "solc_evm_version": "cancun",
    "solc": "solc",
    "optimistic_loop": true,
    "optimistic_fallback": true,
    "loop_iter": "3",
    "rule_sanity": "basic",
    "build_cache": true,
    "server": "production",
    "verify": "YourContract:certora/specs/validation_yourcontract.spec"
}
```

## 7.4 Run Validation

```bash
# Clear cache and run
rm -rf .certora_internal
certoraRun certora/confs/validation_yourcontract.conf

# Check results:
# ❌ ANY FAIL → Fix the gap and re-run
# ✅ ALL PASS → Do NOT proceed to Phase 7 yet.
#              Complete Validation Evidence Review (Section 7.5) first.
```

> **⚠️ STOP: "ALL PASS" ≠ "CORRECTLY PASSED"**  
> A green dashboard is necessary but not sufficient. Proceed to Section 7.5
> to verify the validation is non-degenerate before writing real rules.

## 7.5 Validation Evidence Review 

> **Goal:** Prove that "ALL PASS" means "ALL PASSED CORRECTLY" —  
> not "ALL PASSED VACUOUSLY" or "ALL PASSED DEGENERATELY"

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   THE PROBLEM THIS SOLVES:                                               │
│                                                                          │
│   A junior engineer sees green checkmarks and says "PASSED!"            │
│   But:                                                                   │
│   • satisfy rules found degenerate witnesses (amount=0, balance=0)      │
│   • Ghost sync passed trivially (0 == 0 because hook never fired)       │
│   • Mutation path rule missed a function (incomplete whitelist)          │
│   • rule_sanity: basic didn't catch partial vacuity                     │
│                                                                          │
│   Without evidence review, the real spec is built on sand.              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.5.1 Evidence Collection Template

Add this section to your `{contract}_causal_validation.md`:

```markdown
## VALIDATION EVIDENCE REVIEW

> **Prover Job URL:** [paste URL from certoraRun output]
> **Prover Version:** [version]
> **Date:** [DATE]
> **Reviewer:** [NAME]

---

### Rule Status Table

| # | Rule Name | Status | Witness Quality | Notes |
|---|-----------|--------|-----------------|-------|
| 1 | validation_reachability_function1 | PASS | ✅ Non-degenerate | amount=500, balance changed |
| 2 | validation_reachability_function2 | PASS | ⚠️ Edge case | amount=0 — re-check |
| 3 | validation_revert_reachability_X | PASS | ✅ Non-degenerate | unauthorized caller |
| 4 | validation_mutation_paths_var1 | PASS | ✅ Complete | 3/3 functions covered |
| 5 | validation_ghost_sync | PASS | ✅ Non-trivial | ghost=1500, storage=1500 |

---

### Satisfy Witness Inspection

For EACH satisfy rule, open the Prover output and inspect the witness:

| Satisfy Rule | Witness Values | Non-Degenerate? | Action |
|-------------|---------------|-----------------|--------|
| reachability_function1 | amount=?, balance_before=?, balance_after=? | Yes/No | [ACCEPT / INVESTIGATE] |
| reachability_function2 | amount=?, ... | Yes/No | [ACCEPT / INVESTIGATE] |
| revert_reachability_X | sender=?, ... | Yes/No | [ACCEPT / INVESTIGATE] |

**Non-Degenerate Criteria:**
- ❌ DEGENERATE: amount=0, from=to, balance unchanged, all zeros
- ❌ DEGENERATE: Only one possible witness (over-constrained require)
- ✅ NON-DEGENERATE: Realistic values, meaningful state change
- ✅ NON-DEGENERATE: Multiple possible witnesses exist

**If degenerate:** The satisfy "passed" but proved nothing useful.
Fix: Loosen constraints or add `require amount > 0` style bounds.

---

### Ghost Sync Witness Inspection

For EACH ghost sync rule, verify the ghost is alive:

| Ghost | Pre-Value | Post-Value | Hook Fired? | Non-Trivial? |
|-------|-----------|------------|-------------|-------------|
| sumVariable | [value] | [value] | Yes/No | Yes/No |

**Non-Trivial Criteria:**
- ❌ TRIVIAL: ghost=0 before AND after (hook may never fire)
- ❌ TRIVIAL: ghost == storage only because both are init values
- ✅ NON-TRIVIAL: ghost changes AND matches storage after mutation

**If trivial:** The ghost may be dead (wrong slot, wrong contract binding).
Fix: Verify hook target matches actual storage layout.

---

### Mutation Path Completeness Check

For EACH mutation path rule:

| Variable | Functions in Whitelist | Functions in Phase 0 Entry Points | Missing? |
|----------|----------------------|-----------------------------------|----------|
| var1 | function1, function2 | function1, function2, function3 | function3! |

**Cross-reference against Phase 0 entry points.**
If a function CAN modify the variable but is NOT in the whitelist,
the mutation path rule is incomplete.

---

### Advanced Sanity Run

**Required:** Re-run validation with `"rule_sanity": "advanced"`

```json
// In validation_{contract}.conf, temporarily change:
"rule_sanity": "advanced"  // was: "basic"
```

```bash
certoraRun certora/confs/validation_yourcontract.conf
```

| Sanity Check | Status | Notes |
|-------------|--------|-------|
| rule_not_vacuous (all rules) | PASS/FAIL | |
| invariant_not_trivial_postcondition | PASS/FAIL | |
| [any other flags] | PASS/FAIL | |

**If any advanced sanity check FAILS:** The validation has hidden vacuity.
Fix the modeling gap before proceeding.

---

### Sign-Off

- [ ] All satisfy witnesses inspected and confirmed non-degenerate
- [ ] All ghost sync witnesses inspected and confirmed non-trivial
- [ ] Mutation path whitelists cross-referenced against Phase 0 entry points
- [ ] Advanced sanity run (`rule_sanity: advanced`) completed with no failures
- [ ] Prover job URL recorded for audit trail

**Reviewer Sign-Off:** I have reviewed all witnesses and confirm the
validation infrastructure is sound. Ready to proceed to Phase 7.

**Signed:** [NAME]  
**Date:** [DATE]
```

### 7.5.2 Common Degenerate Witnesses

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| amount = 0 in every satisfy | Over-constrained `require` or missing token setup | Add `require amount > 0` or fix DISPATCHER |
| from == to in transfer satisfy | Prover finds easiest path | Add `require from != to` (or accept and test both) |
| Ghost = 0 before and after | Hook targets wrong slot or wrong contract | Verify hook Sstore target matches actual storage layout |
| Only 1 function in mutation witness | Other functions always revert | Check reachability of missing functions |
| All witnesses use same address | Insufficient address diversity | Add `require addr1 != addr2` where needed |
| Satisfy passes but witness has msg.value > 0 for non-payable | Modeling allows impossible values | Add `require e.msg.value == 0` for non-payable functions |

### 7.5.3 When to STOP vs PROCEED

```
┌─────────────────────────────────────────────────────────────────────────┐
│ STOP — Do NOT proceed to Phase 7:                                       │
│                                                                          │
│ • Any satisfy witness is degenerate AND no realistic witness exists      │
│ • Ghost sync passes trivially (0 == 0) and hook never fires             │
│ • Mutation path whitelist is missing functions from Phase 0              │
│ • Advanced sanity run has ANY failure                                    │
│ • You cannot explain WHY a witness has the values it has                │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│ PROCEED — Phase 7 is safe:                                               │
│                                                                          │
│ • All satisfy witnesses show realistic, non-zero state changes          │
│ • Ghost sync witnesses show ghost values matching storage after mutation │
│ • Mutation path whitelists match Phase 0 entry points exactly           │
│ • Advanced sanity run passes all checks                                  │
│ • Evidence artifact is complete and signed off                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# 8. PHASE 4-6: MODELING & SANITY

## 8.1 Modeling Decisions (Phase 4)

Update your `spec_authoring.md` with final modeling decisions.

## 8.2 Ghost Design (Phase 5)

For each property marked "Aggregate/History Required?: Yes":

| Ghost Pattern | Use When | Template |
|---------------|----------|----------|
| Sum tracking | Total = sum of mapping | `ghost mathint sum` + Sstore hook |
| History | Need previous value | `ghost X prevValue` + hook |
| Counter | Count occurrences | `ghost mathint count` + hook |

## 8.3 Sanity Gate (Phase 6)

**DO NOT proceed until ALL boxes checked:**

```markdown
### Pre-CVL Sanity Gate

**Execution Closure**
- [ ] All entry points enumerated
- [ ] All external reads have modeling decision  
- [ ] All external writes have modeling decision
- [ ] All NONDET justified (doesn't affect invariant)

**Causal Closure**
- [ ] validation spec written
- [ ] Function reachability (`satisfy`) rules PASS for all entry points ← NEW v1.8
- [ ] Failure-path reachability (`satisfy lastReverted`) rules PASS for critical revert conditions ← NEW v1.9
- [ ] certoraRun validation PASSED (all rules)
- [ ] All ghosts have complete hooks
- [ ] init_state axioms for all ghosts

**Invariant Dependency Safety**
- [ ] Every invariant annotated with `@dev Level: N` (dependency depth)
- [ ] Dependency DAG sketched in `causal_validation.md` (no cycles)
- [ ] Base invariants (Level 1) proven in isolation first (`--rule`)
- [ ] Higher-level invariants proven only after their dependencies pass

**Custom Summary Accuracy** 
- [ ] For each custom summary: behavior documented (overapproximation vs exact)
- [ ] Custom summaries do not assume determinism the real function doesn't guarantee
- [ ] Custom summary accuracy justified in `spec_authoring.md`

**Validation Evidence Review** 
- [ ] Prover job URL recorded in `causal_validation.md`
- [ ] All satisfy witnesses inspected — confirmed non-degenerate
- [ ] All ghost sync witnesses inspected — confirmed non-trivial (ghost ≠ 0)
- [ ] Mutation path whitelists cross-referenced against Phase 0 entry points
- [ ] Re-run with `rule_sanity: advanced` completed — no failures
- [ ] Evidence sign-off completed in `causal_validation.md`

**Bounded State**
- [ ] Array params: require arr.length < 100
- [ ] Timestamps: require e.block.timestamp < 2^40
- [ ] Balances: realistic bounds if needed

**Property Quality**
- [ ] Each property has one truth owner (or explicitly modeled)
- [ ] No property requires honest admin
- [ ] Every property maps to real exploit
```

---

# 9. PHASE 7: WRITE CVL

> **You may only enter this phase after Phase 6 sanity gate PASSES**

## 9.0 Transition from Validation Spec to Real Spec

When your validation spec PASSES, you've proven your **infrastructure is correct**. Now create the real spec by copying and modifying.

> **⚠️ PREREQUISITE:** Before entering Phase 7, you MUST have completed the
> Validation Evidence Review (Section 7.5). "ALL PASS" alone is not sufficient.
> Evidence of non-degenerate witnesses and advanced sanity must be recorded.

### What Validation Passing Guarantees

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   VALIDATION PASSED = Your INFRASTRUCTURE is correct                    │
│                                                                          │
│   ✅ ELIMINATED (you won't see these bugs):                             │
│   ├── Always-reverting functions (vacuity) — caught by satisfy rules    │
│   ├── Ghost desynchronization                                           │
│   ├── Missing mutation paths (incomplete hooks)                         │
│   ├── Wrong hook types                                                  │
│   ├── Missing init_state axioms                                         │
│   ├── DISPATCHER/NONDET misconfiguration                                │
│   └── Method signature mismatches                                       │
│                                                                          │
│   ⚠️ STILL POSSIBLE (but easy to diagnose):                             │
│   ├── Logic errors in invariant/rule (wrong operator, wrong var)        │
│   ├── Missing preconditions (forgot requireInvariant)                   │
│   ├── Property too strong (not actually true)                           │
│   └── REAL CONTRACT BUG (this is what you want to find!)            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step: Create Real Spec from Validation

```bash
# Step 1: Copy validation spec as starting point
cp certora/specs/validation_{target}.spec certora/specs/{Target}.spec

# Step 2: Copy validation conf
cp certora/confs/validation_{target}.conf certora/confs/{Target}.conf

# Step 3: Update conf to point to real spec
# Change: "verify": "Contract:certora/specs/{Target}.spec"
```

### What to KEEP, DELETE, ADD

| Component | Action | Reason |
|-----------|--------|--------|
| `methods { }` block | **KEEP** | Proven correct |
| `ghost` declarations | **KEEP** | Proven synchronized |
| `hook` definitions | **KEEP** | Proven complete |
| `using` statements | **KEEP** | Contract bindings work |
| Helper functions | **KEEP** | Utility functions |
| `rule validation_reachability_*` | **DELETE** | Reachability proven; no longer needed |
| `rule validation_*` | **DELETE** | Replace with real properties |
| Real `invariant` | **ADD** | From candidate_properties.md |
| Real `rule` | **ADD** | From candidate_properties.md |

### Template: Real Spec Header

```cvl
/*
 * ═══════════════════════════════════════════════════════════════
 * [CONTRACT_NAME] VERIFICATION SPEC
 * ═══════════════════════════════════════════════════════════════
 * Contract: [ContractName].sol
 * Author: [Name]
 * Date: [Date]
 * 
 * Validation: PASSED ✅
 * Infrastructure copied from: validation_{target}.spec
 * ═══════════════════════════════════════════════════════════════
 */
```

---

## 9.1 Spec Structure

```cvl
/*
 * ═══════════════════════════════════════════════════════════════
 * [CONTRACT_NAME] VERIFICATION SPEC
 * ═══════════════════════════════════════════════════════════════
 * Contract: [ContractName].sol
 * Author: [Name]
 * Date: [Date]
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
    // ─────────────────────────────────────────────────────────────
    // Contract Functions
    // ─────────────────────────────────────────────────────────────
    function deposit(uint256 amount) external;
    function withdraw(uint256 amount) external;
    
    // ─────────────────────────────────────────────────────────────
    // View Functions (envfree)
    // ─────────────────────────────────────────────────────────────
    function balanceOf(address user) external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
    
    // ─────────────────────────────────────────────────────────────
    // External Contracts - DISPATCHER
    // ─────────────────────────────────────────────────────────────
    function _.transfer(address, uint256) external => DISPATCHER(true);
    function _.transferFrom(address, address, uint256) external => DISPATCHER(true);
    function _.balanceOf(address) external => DISPATCHER(true);
    
    // ─────────────────────────────────────────────────────────────
    // Unrelated Calls - NONDET (justified)
    // ─────────────────────────────────────────────────────────────
    // [Contract].[function] - does not affect [invariant] because [reason]
    function _.unrelatedFunction() external => NONDET;
}

// ═══════════════════════════════════════════════════════════════
// GHOSTS
// ═══════════════════════════════════════════════════════════════

/// @title Sum of all user balances
/// @notice Used for totalSupply consistency invariant
ghost mathint sumBalances {
    init_state axiom sumBalances == 0;
}

// ═══════════════════════════════════════════════════════════════
// HOOKS
// ═══════════════════════════════════════════════════════════════

/// @notice Update sum when any balance changes
hook Sstore currentContract._balances[KEY address user] uint256 newBal 
    (uint256 oldBal) {
    sumBalances = sumBalances + newBal - oldBal;
}

// ═══════════════════════════════════════════════════════════════
// HELPER FUNCTIONS
// ═══════════════════════════════════════════════════════════════

/// @notice Bundle all invariants for rule preconditions
function validState() {
    requireInvariant solvency();
    requireInvariant totalEqualsSum();
    requireInvariant stateValid();
}

/// @notice Standard environment constraints
function validEnv(env e) {
    require e.msg.sender != 0;
    require e.msg.sender != currentContract;
    require e.block.timestamp > 0;
    require e.block.timestamp < 2^40;
}

// ═══════════════════════════════════════════════════════════════
// INVARIANTS (ordered by dependency - Level 1 first)
// ═══════════════════════════════════════════════════════════════

/// @title State enum is always valid
/// @notice Level 1 - no dependencies
invariant stateValid()
    getState() <= 2

/// @title Total supply equals sum of balances
/// @notice Level 2 - depends on ghost
invariant totalEqualsSum()
    to_mathint(totalSupply()) == sumBalances
    {
        preserved {
            requireInvariant stateValid();
        }
    }

/// @title Contract is always solvent
/// @notice Level 3 - depends on totalEqualsSum
invariant solvency()
    nativeBalances[currentContract] >= totalOwed()
    {
        preserved {
            requireInvariant stateValid();
            requireInvariant totalEqualsSum();
        }
    }

// ═══════════════════════════════════════════════════════════════
// RULES
// ═══════════════════════════════════════════════════════════════

/// @title Deposit increases user balance
/// @notice Verifies correct deposit behavior
rule deposit_increasesBalance(address user, uint256 amount) {
    // Preconditions
    validState();
    env e;
    validEnv(e);
    require e.msg.sender == user;
    require amount > 0;
    
    // Capture before
    uint256 balBefore = balanceOf(user);
    
    // Action
    deposit(e, amount);
    
    // Postcondition
    uint256 balAfter = balanceOf(user);
    
    assert balAfter > balBefore, "Deposit should increase balance";
}

/// @title Only owner can change settings
/// @notice Access control verification
rule onlyOwner_canChangeSettings(method f, address caller)
    filtered {
        f -> f.selector == sig:setParameter(uint256).selector
    }
{
    validState();
    env e;
    require e.msg.sender == caller;
    
    uint256 paramBefore = getParameter();
    
    f(e, _);  // Execute the filtered function
    
    uint256 paramAfter = getParameter();
    
    assert paramBefore != paramAfter => caller == owner(),
        "Only owner should change settings";
}
```

## 9.2 Create `{Contract}.conf`

```json
{
    "files": [
        "contracts/YourContract.sol",
        "contracts/Dependency.sol",
        "certora/harnesses/DummyToken.sol"
    ],
    "link": [
        "YourContract:TOKEN=DummyToken"
    ],
    "msg": "[ContractName] Verification",
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
    "verify": "YourContract:certora/specs/YourContract.spec"
}
```

---

# 10. RUNNING & DEBUGGING

## 10.1 Run Commands

```bash
# ═══════════════════════════════════════════════════════════════
# Clear cache (do this when changing spec structure)
# ═══════════════════════════════════════════════════════════════
rm -rf .certora_internal

# ═══════════════════════════════════════════════════════════════
# Run validation (ALWAYS FIRST)
# ═══════════════════════════════════════════════════════════════
certoraRun certora/confs/validation_yourcontract.conf

# ═══════════════════════════════════════════════════════════════
# Run real spec
# ═══════════════════════════════════════════════════════════════
certoraRun certora/confs/YourContract.conf

# ═══════════════════════════════════════════════════════════════
# Run specific rule only
# ═══════════════════════════════════════════════════════════════
certoraRun certora/confs/YourContract.conf --rule "deposit_increasesBalance"

# ═══════════════════════════════════════════════════════════════
# Run with output capture
# ═══════════════════════════════════════════════════════════════
certoraRun certora/confs/YourContract.conf 2>&1 | tee prover_output.log
```

## 10.2 Common Compilation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `X is not a valid EVM type` | Enum/custom type in hook | Use `Solidity.Type` or underlying type |
| `already declared in scope` | Name conflict | Rename your CVL function |
| `could not find method` | Wrong signature | Check exact signature in contract |
| `Type mismatch in hook` | Hook type ≠ storage type | Match Solidity types exactly |
| `NONDET not allowed` | NONDET on state-changing | Use DISPATCHER instead |

## 10.3 Counterexample Debugging

When a rule FAILS, use `certora-ce-diagnosis-framework.md` (enhanced with Tutorial Lesson 02 workflow):

**5-Step Investigation Process (from BEST_PRACTICES Section 2):**
1. Run entire spec first (get overview of failures)
2. Focus on one rule (`--rule rule_name`)
3. Analyze call trace (storage, arguments, returns)
4. Identify deviation (spec bug vs real bug)
5. Fix and document

**Call Trace Analysis:**

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Is the CE showing a REAL bug or SPURIOUS result?    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Check the call trace:                                        │
│ - Does it use realistic values?                              │
│ - Does it exploit HAVOC on external calls?                   │
│ - Does it violate implicit assumptions?                      │
│                                                              │
│ REAL BUG:                                                    │
│ - Values are realistic                                       │
│ - No HAVOC exploitation                                      │
│ - Represents actual attack vector                            │
│ → FIX THE CONTRACT                                           │
│                                                              │
│ SPURIOUS:                                                    │
│ - Unrealistic values (e.g., balance > total supply)          │
│ - HAVOC changed external state unexpectedly                  │
│ - Missing modeling constraint                                │
│ → FIX THE SPEC                                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 10.4 Performance Optimization & Timeout Mitigation

> **NEW in v1.4:** See **advanced-cli-reference.md** for complete guide

When rules timeout or run slowly, use these strategies:

### Quick Timeout Fixes

| Problem | Solution | Command |
|---------|----------|---------|
| Multiple slow rules | Split to separate jobs | `--split_rules "pattern"` |
| Complex assertions | Check separately | `--multi_assert_check` |
| Loop complexity | Adjust iterations | `--loop_iter N` |
| High path count | Control flow splitting | See advanced guide |

### Common Performance Commands

```bash
# Split heavy rules (gives each more resources)
certoraRun certora/confs/Contract.conf --split_rules "solvency_*" "invariant_*"

# Multi-assert check (timeout mitigation)
certoraRun certora/confs/Contract.conf --multi_assert_check

# Loop handling
certoraRun certora/confs/Contract.conf --loop_iter 3

# Control flow splitting (eager splitting for large code)
certoraRun certora/confs/Contract.conf \
    --prover_args '-smt_initialSplitDepth 5 -depth 15'

# Multiple counterexamples for debugging
certoraRun certora/confs/Contract.conf --rule failing_rule --multi_example
```

### Performance Decision Tree

```
Rule timing out?
│
├─► Multiple slow rules? → --split_rules
├─► Complex assertions? → --multi_assert_check
├─► Loops in contract? → --loop_iter N (start with 1-3)
├─► Large source code? → --prover_args '-smt_initialSplitDepth 5'
└─► Still timing out? → See advanced-cli-reference.md Section 1
```

### Advanced Debugging Flags

```bash
# Multiple counterexamples (see different failure paths)
certoraRun config.conf --rule failing_rule --multi_example

# Independent satisfy (check each satisfy separately)
certoraRun config.conf --independent_satisfy

# Rule sanity (ensure non-vacuous)
certoraRun config.conf --rule_sanity basic

# Coverage analysis (find gaps)
certoraRun config.conf --coverage_info advanced
```

**→ For detailed strategies, loop handling, multi-version projects, and harness patterns:**  
**See advanced-cli-reference.md**

---

# 11. TEMPLATES

## 11.1 Harness Template (DummyToken.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @title DummyToken
 * @notice Simplified ERC20 for Certora verification
 * @dev Removes complex logic that causes timeouts
 */
contract DummyToken {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
}
```

## 11.2 Common.spec Template

```cvl
/*
 * Common definitions shared across specs
 */

// Standard address constraints
function validAddress(address a) returns bool {
    return a != 0;
}

// Standard env constraints  
function validEnv(env e) returns bool {
    return e.msg.sender != 0 && 
           e.block.timestamp > 0 && 
           e.block.timestamp < 2^40;
}

// Max values
definition MAX_UINT256() returns uint256 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff;
definition MAX_UINT128() returns uint256 = 0xffffffffffffffffffffffffffffffff;
```

---

# 12. QUICK REFERENCE

## 12.1 Command Cheat Sheet

```bash
# Setup
mkdir -p spec_authoring certora/{specs,confs,harnesses,helpers}

# Analysis
grep -n "function.*external" contracts/Contract.sol    # Entry points
grep -n "^\s*mapping\|^\s*uint" contracts/Contract.sol # Storage
grep -n "\.[a-zA-Z]*(" contracts/Contract.sol          # External calls

# Run
rm -rf .certora_internal                               # Clear cache
certoraRun certora/confs/validation.conf               # Validation
certoraRun certora/confs/Contract.conf                 # Real spec
certoraRun config.conf --rule "ruleName"               # Single rule
```

## 12.2 CVL Syntax Quick Reference

```cvl
// Methods block
function name(args) external;                          // Declare
function name(args) external returns (T) envfree;      // Envfree view
function _.name(args) external => DISPATCHER(true);    // External dispatch
function _.name(args) external => NONDET;              // Non-deterministic

// Ghost
ghost mathint myGhost { init_state axiom myGhost == 0; }

// Hook (must match Solidity type)
hook Sstore contract.var[KEY address k] uint256 new (uint256 old) { ... }

// Invariant
invariant myInvariant() condition { preserved { requireInvariant other(); } }

// Rule
rule myRule(args) filtered { f -> condition } { ... assert condition; }

// Revert verification (NEW v1.6)
f@withrevert(e, args);                   // Include revert paths in analysis
bool success = !lastReverted;             // Capture revert status IMMEDIATELY
assert success <=> (preconditions);       // Liveness: success iff conditions met
assert lastReverted <=> (revert_causes);  // Exhaustive revert enumeration
```

## 12.3 File Checklist

Before running prover, verify:

- [ ] `spec_authoring/{contract}_spec_authoring.md` - Phases 0, -1, 4, 5, 6 complete
- [ ] `spec_authoring/{contract}_candidate_properties.md` - All properties listed
- [ ] `spec_authoring/{contract}_causal_validation.md` - Mutation paths documented
- [ ] `certora/specs/validation_{contract}.spec` - Validation rules
- [ ] `certora/confs/validation_{contract}.conf` - Validation config
- [ ] Validation run PASSED ✓
- [ ] `certora/specs/{Contract}.spec` - Real spec
- [ ] `certora/confs/{Contract}.conf` - Real config
- [ ] Required harnesses in `certora/harnesses/`

## 12.4 When Things Go Wrong

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Compilation error | Wrong syntax/types | Check CVL 2.0 docs |
| Timeout | Too complex | Simplify with harnesses |
| Spurious CE (HAVOC) | Missing modeling | Add DISPATCHER |
| Spurious CE (values) | Missing constraint | Add realistic bounds |
| Invariant fails on all | Ghost not synced | Check hooks |
| Rule fails unexpectedly | Missing validState() | Add requireInvariant |

**NEW:** See `best-practices-from-certora.md` Section 6 for common pitfalls and anti-patterns.

---

# FINAL CHECKLIST

Before considering verification complete:

```
□ Phase 0-1: Execution reality fully mapped
□ Phase 2: All security properties discovered
□ Phase 2.5: Each property classified (INVARIANT/RULE)
□ Phase 3.5: Causal validation PASSED
□ Phase 3.5: Function reachability (satisfy) PASSED for all entry points  
□ Phase 3.5: Validation Evidence Review completed — witnesses non-degenerate  
□ Phase 3.5: Advanced sanity run (rule_sanity: advanced) — no failures  
□ Phase 6: Sanity gate ALL CHECKED
□ Phase 7: CVL spec written
□ Prover: All rules PASS
□ Review: No hidden trust assumptions
□ Documentation: Decisions logged in spec_authoring.md
□ Revert Coverage: Every state-changing function has @withrevert verification  ← NEW v1.6
□ Liveness Assertions: Use <=> (biconditional) not just => (implication)    ← NEW v1.6
□ No Silent Revert Pruning: No rule relies on default revert-ignore behavior ← NEW v1.6
```

---

# 13. QUICK START CHAT PROMPTS

> **Use this section when starting or continuing verification with an AI assistant.**  
> **Copy the appropriate prompt below and paste it into your chat.**

## 13.1 For a Brand New Verification Project

```markdown
I am starting a formal verification project using Certora for the following contract:

**Project Location:** [/path/to/project]
**Primary Target Contract:** [ContractName.sol at path/to/contract]
**Contract Dependencies:** [List the files that the target imports]
**Token Standard (if any):** [ERC-20 / ERC-721 / ERC-2771 / WETH / None]
**Uses unchecked{} or assembly?** [Yes/No]  ← NEW v1.7

Please help me follow the certora-master-guide.md workflow:

1. First, create the folder structure (spec_authoring/, certora/specs/, certora/confs/, certora/harnesses/)
2. Create the analysis documents for this target:
   - {target}_spec_authoring.md
   - {target}_candidate_properties.md  
   - {target}_causal_validation.md
3. Begin Phase 0: Analyze the target contract to extract:
   - All entry points (external/public functions)
   - All storage variables
   - All external calls
   - All modifiers/access control
   - All `unchecked{}` blocks and type casts ← NEW v1.7
4. Run Phase 0 Builtin Safety Scan (if Prover v8.8.0+):  ← NEW v1.7
   - `use builtin rule uncheckedOverflow;` to detect unchecked arithmetic
   - `use builtin rule safeCasting;` to detect unsafe type narrowing
   - Triage results before writing any custom rules

The framework documents are already in my project root.

**Key references to use throughout (v1.9):**
- cvl-language-deep-dive.md — Complete CVL language reference (types, ghosts, hooks, operators, builtin rules §19.1)
- verification-playbooks.md — Worked examples for ERC-20, WETH, ERC-721 + Phase 0 builtin scan
- best-practices-from-certora.md — Sections 7-9 (vacuity defense, requireInvariant lifecycle, edge cases)
- best-practices-from-certora.md — §8.4 Circular dependency detection for invariant DAGs  ← NEW v1.9
- certora-spec-framework.md — Revert/failure-path patterns (Pattern B: @withrevert PREFERRED)
- certora-spec-framework.md — Custom summary accuracy protocol & invariant DAG protocol  ← NEW v1.9
- categorizing-properties.md — MUST REVERT WHEN checklist for property discovery
```

## 13.2 For Continuing Phase 0 / Phase -1

```markdown
Continue the Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** [0 / -1]

Please analyze the contract and help me fill in the spec_authoring document:
- Phase 0: Entry points, storage variables, asset flows
- Phase 0: Identify all `unchecked{}` blocks and type-narrowing casts  ← NEW v1.7
- Phase 0: Run builtin safety scan (`uncheckedOverflow`, `safeCasting`)  ← NEW v1.7
- Phase -1: External contracts, interaction ownership table, modeling decisions
- Phase -1: For each external call, document revert conditions  ← NEW v1.6

Reference: 
- certora-master-guide.md sections 3 and 4
- best-practices-from-certora.md Section 4 (harness patterns if needed)
- cvl-language-deep-dive.md §19.1 (builtin rules reference table)  ← NEW v1.7
- spec-authoring-certora.md Phase 3 (revert/failure behavior field)  ← NEW v1.6
```

## 13.3 For Phase 2 (Property Discovery)

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** 2 (Property Discovery)

Based on the Phase 0/-1 analysis, help me discover security properties using categorizing-properties.md:
- Valid States (range constraints)
- State Transitions (function effects)
- System-Level (aggregates, sums)
- Access Control (who can do what)
- **Revert Behavior (MUST REVERT WHEN)**  ← NEW v1.6

**Apply these techniques:**
- Property prioritization (HIGH/MEDIUM/LOW) - categorizing-properties.md Section 7
- Dual mindset (Should Always / Should Never) - Section 5
- Test mining (extract from existing tests) - Section 6
- Avoid the 4 fatal mistakes - BEST_PRACTICES Section 1

**For each function, consider the Liveness/Effect/No-Side-Effect triple:**
- Liveness: `assert success <=> (preconditions)` — use biconditional, not implication
- Effect: `assert success => (state_changes)`
- No Side Effect: `assert uninvolved_state unchanged`
See verification-playbooks.md Section 4 and cvl-language-deep-dive.md Section 15.

**v1.6 — For each state-changing function, also enumerate revert conditions:**
- Use the MUST REVERT WHEN checklist (categorizing-properties.md)
- Document: What caller restrictions cause revert?
- Document: What input validation causes revert?
- Document: What state preconditions cause revert?
- Fill in the REVERT field in each property template

Output should go into: spec_authoring/{target}_candidate_properties.md
```

## 13.3.1 For Phase 2 with Dual Mindset + Test Mining

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Test Files:** [path/to/tests/]
**Current Phase:** 2 (Property Discovery - Comprehensive)

Please help me discover properties using the DUAL MINDSET approach:

**1. Mine existing tests:**
   - Extract implicit invariants from test assertions
   - Identify threat assumptions
   - Note blind spots (what's NOT tested)
   - Check: do tests cover revert paths? (vm.expectRevert)  ← NEW v1.6

**2. RIGHT/WRONG behavior enumeration:**
   - For each critical function, document:
     - SHOULD ALWAYS: "When X, Y should always happen"
     - SHOULD NEVER: "Even if X, Y must never happen"
     - MUST REVERT WHEN: "If X, the call must revert"  ← NEW v1.6

**3. Categorize all properties:**
   - Valid States / State Transitions / System-Level / Threat-Driven / Revert Behavior

Reference:
- categorizing-properties.md sections 5 and 6 (Dual Checklist & Test Mining)
- categorizing-properties.md MUST REVERT WHEN checklist (6 questions)  ← NEW v1.6
```

## 13.4 For Phase 3.5 (Causal Validation)

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** 3.5 (Causal Validation)

Create the validation spec and conf to verify mutation paths are complete:
1. Create certora/specs/validation_{target}.spec
2. Create certora/confs/validation_{target}.conf
3. **Write `satisfy` reachability rules for EVERY state-changing function**  ← NEW v1.8
   - `f@withrevert(e, args); satisfy !lastReverted;`
   - If any satisfy is VIOLATED → function always reverts → all asserts are vacuous
   - Fix harness / DISPATCHER / require constraints before proceeding
4. **Write `satisfy lastReverted` rules for critical revert conditions**  ← NEW v1.9
   - Proves revert paths are reachable (guards biconditional `<=>` from vacuity)
5. Include validation rules for each INVARIANT variable
6. Include ghost synchronization tests if ghosts are needed
7. Include revert validation: for each state-changing function, write a  ← NEW v1.6
   `@withrevert` rule confirming revert conditions are exhaustive
8. If using Prover v8.8.0+, include `use builtin rule sanity;` to catch  ← NEW v1.7
   vacuous rules early
9. **Annotate invariants with `@dev Level: N` and document dependency DAG**  ← NEW v1.9
   - Prove base invariants (Level 1) first, then higher levels in order
10. **Validate custom summary accuracy** (if using custom summaries)  ← NEW v1.9
    - Document whether each summary is exact, overapproximation, or underapproximation

**Validation Execution Order:**
- Run reachability (`satisfy`) rules FIRST → proves functions are live
- Run failure-path reachability (`satisfy lastReverted`) → proves revert paths are live  ← NEW v1.9
- Then run mutation path rules → proves modeling is complete
- Then run ghost sync rules → proves ghosts track reality
- Only after ALL PASS → complete Validation Evidence Review (Section 7.5)  ← NEW v2.0
- Only after Evidence Review signed off → proceed to Phase 7 (real spec)  ← NEW v2.0

Reference:
- certora-master-guide.md section 7 (validation spec template with Rule 0)
- cvl-language-deep-dive.md Sections 8-9 (ghost declaration, init_state axiom, hook syntax)
- cvl-language-deep-dive.md §19.1 (builtin rules — sanity, deepSanity)  ← NEW v1.7
- best-practices-from-certora.md Section 7 (vacuity defense, satisfy for reachability, failure-path satisfy)  ← updated v1.9
- best-practices-from-certora.md Section 8 (require → requireInvariant lifecycle)
- best-practices-from-certora.md §8.4 (circular dependency detection)  ← NEW v1.9
- certora-spec-framework.md Revert/Failure-Path Checklist  ← NEW v1.6
- certora-spec-framework.md Invariant DAG protocol & custom summary accuracy  ← NEW v1.9
```

## 13.4.1 For Validation Evidence Review (Validation ALL PASS → Verify Quality)  ← NEW v2.0

```markdown
My validation spec shows ALL PASS for [ContractName]. Before writing the real
spec, I need to verify the validation actually passed correctly (not vacuously
or degenerately).

**Target:** [path/to/ContractName.sol]
**Validation Spec:** certora/specs/validation_{target}.spec
**Prover Job URL:** [paste URL]
**Validation Document:** spec_authoring/{target}_causal_validation.md

Please help me complete the Validation Evidence Review:

1. **Satisfy Witness Inspection:**
   - For each satisfy rule, inspect the Prover witness (counterexample)
   - Verify witnesses are NON-DEGENERATE:
     - ❌ Degenerate: amount=0, from==to, all balances zero, unchanged state
     - ✅ Non-degenerate: realistic values, meaningful state changes
   - If any witness is degenerate, investigate whether a realistic witness exists

2. **Ghost Sync Witness Inspection:**
   - For each ghost sync rule, verify the ghost value is NON-TRIVIAL:
     - ❌ Trivial: ghost=0 before AND after (hook may never fire)
     - ✅ Non-trivial: ghost changes AND matches storage after mutation
   - If trivial, check hook target matches actual storage layout

3. **Mutation Path Completeness:**
   - Cross-reference each mutation path rule's function whitelist against
     the Phase 0 entry points table
   - Identify any functions that CAN modify the variable but are NOT in the whitelist
   - If any missing, the mutation path model is incomplete

4. **Advanced Sanity Run:**
   - Re-run with `"rule_sanity": "advanced"` in the .conf
   - Check for any `rule_not_vacuous` or `invariant_not_trivial_postcondition` failures
   - If any fail, the validation has hidden vacuity

5. **Evidence Sign-Off:**
   - Fill in the Validation Evidence Review template in causal_validation.md
   - Record Prover job URL, rule status table, witness inspections
   - Produce signed completion statement

**Gate Criteria (ALL must be true to proceed to Phase 7):**
- All satisfy witnesses confirmed non-degenerate
- All ghost sync witnesses confirmed non-trivial
- Mutation path whitelists match Phase 0 entry points
- Advanced sanity run shows no failures
- Evidence artifact complete and signed

Reference:
- certora-master-guide.md section 7.5 (Validation Evidence Review template)
- certora-master-guide.md section 7.5.2 (Common Degenerate Witnesses table)
- certora-master-guide.md section 7.5.3 (STOP vs PROCEED decision matrix)
- cvl-language-deep-dive.md §19 (Invariant Sanity Checks — rule_not_vacuous)
```

## 13.5 For Phase 7 (Validation PASSED → Write Real Spec)

```markdown
My validation spec PASSED for [ContractName]. Ready to write the real spec.

**Target:** [path/to/ContractName.sol]
**Token Standard (if any):** [ERC-20 / ERC-721 / WETH / None]
**Validation Spec:** certora/specs/validation_{target}.spec (PASSED ✅)
**Validation Evidence Review:** COMPLETED ✅  ← NEW v2.0
**Candidate Properties:** spec_authoring/{target}_candidate_properties.md

Please help me create the real spec:
1. Copy infrastructure from validation spec (methods, ghosts, hooks)
2. DELETE all validation_* rules
3. ADD real invariants and rules from candidate_properties.md
4. Use the Liveness/Effect/No-Side-Effect pattern for each function rule
5. Add standard `definition` blocks (nonpayable, nonzerosender, balanceLimited)
6. Use `requireInvariant` (not raw `require`) for proven invariant preconditions
7. For EVERY state-changing function, write @withrevert rules  ← NEW v1.6
   using Pattern B (Complete with biconditional <=>) as the PREFERRED pattern
8. Add `use builtin rule uncheckedOverflow;` and/or `safeCasting;`  ← NEW v1.7
   if the contract uses unchecked{} blocks or type-narrowing casts
9. **Annotate every invariant with `@dev Level: N | Dependencies: ...`**  ← NEW v1.9
   and prove in level order (Level 1 first, then Level 2, etc.)
10. **For each custom summary, add accuracy annotation (Exact/Over/Under)**  ← NEW v1.9
11. Create certora/specs/{Contract}.spec
12. Create certora/confs/{Contract}.conf

Reference:
- certora-master-guide.md section 9.0 (Transition from Validation to Real Spec)
- cvl-language-deep-dive.md (complete CVL reference — types, operators, ghosts, hooks, definitions)
- cvl-language-deep-dive.md §19.1 (builtin rules — uncheckedOverflow, safeCasting)  ← NEW v1.7
- verification-playbooks.md (if ERC-20/721/WETH — follow the complete worked example)
- certora-spec-framework.md (CVL syntax patterns + Revert/Failure-Path Checklist)  ← updated v1.6
- certora-spec-framework.md Pattern B: Complete with @withrevert (PREFERRED)  ← NEW v1.6
- certora-spec-framework.md Invariant DAG protocol & custom summary accuracy  ← NEW v1.9
- best-practices-from-certora.md Sections 3, 7-9 (invariant patterns, vacuity defense, lifecycle, edge cases)
- best-practices-from-certora.md §8.4 (circular dependency detection)  ← NEW v1.9
- quick-reference-v1.3.md (keep open for syntax lookup)
```

## 13.5.1 For Token Standard Verification (ERC-20 / ERC-721 / WETH)

```markdown
I need to verify a [ERC-20 / ERC-721 / WETH] token contract:

**Target:** [path/to/TokenContract.sol]
**Standard:** [ERC-20 / ERC-721 / WETH]
**Prover Version:** [v8.8.0+ / older]  ← NEW v1.7
**Non-standard features:** [mint/burn access control, fee-on-transfer, rebasing, etc.]
**Uses unchecked{} or assembly?** [Yes/No]  ← NEW v1.7

Please use the verification-playbooks.md as the primary reference:

**Phase 0 — Builtin Safety Scan (do this FIRST if Prover v8.8.0+):**  ← NEW v1.7
- Run `use builtin rule uncheckedOverflow;` — tokens often use unchecked{} for gas savings
- Run `use builtin rule safeCasting;` — catch silent truncation in balance/amount casts
- Triage findings before proceeding to custom rules

**For ERC-20:** Follow Section 1 (22-rule playbook with 4-phase methodology):
- Phase 1: Function correctness (transfer, transferFrom, approve, mint, burn)
- Phase 2: No side effects on uninvolved accounts
- Phase 3: Global invariants (totalSupply == sumOfBalances, individual cap, zero-address)
- Phase 4: Authorization (only mint/burn change supply, only transfer changes balances)
- **Every function rule must use @withrevert + biconditional <=>**  ← NEW v1.6

**For ERC-721:** Follow Section 3 (OpenZeppelin pattern):
- Create harness with unsafeOwnerOf/unsafeGetApproved (non-reverting getters)
- Create ERC721ReceiverHarness for DISPATCHER callback resolution
- Use helperSoundFnCall for partially parametric rules
- Handle mint/burn/transferFrom with ownership tracking ghost + hook
- **Verify safeMint/safeTransferFrom callback revert behavior**  ← NEW v1.6
  (See verification-playbooks.md §3.11-3.12)

**For WETH:** Follow Section 2 (Solady pattern):
- Prove solvency invariant: nativeBalances[currentContract] >= totalSupply()
- Use persistent ghost + CALL hook for assembly verification
- Exclude self-calls: require e.msg.sender != currentContract

Also reference:
- cvl-language-deep-dive.md (mathint, satisfy, <=>, @withrevert, persistent ghost, definitions)
- cvl-language-deep-dive.md §19.1 (builtin rules for overflow/casting)  ← NEW v1.7
- best-practices-from-certora.md Section 9 (self-transfer edge case)
- certora-spec-framework.md Pattern B: @withrevert (PREFERRED)  ← NEW v1.6
```

## 13.6 For Debugging Counterexamples

```markdown
I have a failing rule in my Certora verification:

**Target:** [ContractName]
**Failing Rule:** [rule name]
**Error/CE Summary:** [paste the counterexample or error]
**Ghost variables involved (if any):** [ghost names]
**Rule uses @withrevert?** [Yes/No]  ← NEW v1.6

Please help me diagnose using the systematic approach:
1. Is this a REAL bug or SPURIOUS result?
2. If spurious, what modeling is missing?
3. If real, what's the attack vector?
4. If ghost values look wrong, is it a havocing issue?
5. Is this a SILENT PASS (rule passes but revert paths were pruned)?  ← NEW v1.6
   - Does the rule use @norevert (default) and silently skip failure cases?
   - Would adding @withrevert reveal a hidden revert path?
   - Is an implication (=>) masking a missing biconditional (<=>)?

Reference:
- certora-ce-diagnosis-framework.md (comprehensive 5-phase diagnosis + ghost havocing guide)
- certora-ce-diagnosis-framework.md SILENT PASS classification  ← NEW v1.6
- best-practices-from-certora.md Section 2 (5-step investigation workflow from Tutorial Lesson 02)
- cvl-language-deep-dive.md Section 4 (vacuous truth — is the rule trivially passing?)
- cvl-language-deep-dive.md Section 8 (ghost havocing — when/why ghosts get arbitrary values)
- Focus on call trace analysis: storage changes, arguments, return values
```

## 13.6.1 For Loop Timeouts or Inductive Invariant Failures

```markdown
I'm experiencing [timeout / invariant doesn't hold in preserved block] related to loops:

**Target:** [ContractName]
**Problematic Function:** [function with loop]
**Issue:** [timeout / invariant fails / optimistic_loop insufficient]

Please help me resolve using loop handling strategies:
1. Should I use --loop_iter with specific bound?
2. Should I use --optimistic_loop?
3. Do I need loop invariants in preserved block?
4. Should I simplify the loop in a harness?

Reference: best-practices-from-certora.md Section 5 (Loop Handling from Tutorial Lesson 11)
```

## 13.7 Essential Information to Provide

When starting any verification conversation, always include:

| Required Info | Example |
|---------------|---------|
| **Project path** | `/home/user/my-protocol` |
| **Target contract** | `contracts/core/Vault.sol` |
| **Contract name** | `Vault` (as declared in Solidity) |
| **Dependencies** | `imports Token.sol, Oracle.sol, Utils.sol` |
| **Current phase** | Phase 0 / -1 / 2 / 2.5 / 3.5 / 7 |
| **Token standard** | ERC-20 / ERC-721 / WETH / None |
| **Prover version** | v8.8.0+ / older ← NEW v1.7 |
| **unchecked{}/assembly?** | Yes / No ← NEW v1.7 |
| **Invariant dependencies?** | Complex chain / Simple / None ← NEW v1.9 |
| **Custom summaries?** | Yes / No ← NEW v1.9 |

**Optional but helpful:**
- Known external integrations (ERC20, Chainlink, Uniswap, etc.)
- Special patterns (proxy, upgradeable, diamond)
- Existing tests or known issues
- Non-standard features (fee-on-transfer, rebasing, computed storage slots)
- Assembly usage (low-level calls, inline assembly)
- Whether the contract uses `unchecked{}` blocks (triggers builtin overflow scan)  ← NEW v1.7
- Type-narrowing casts like `uint128(x)` (triggers builtin safeCasting scan)  ← NEW v1.7
- Whether invariants form complex dependency chains (triggers DAG protocol)  ← NEW v1.9
- Whether custom function summaries are used (triggers accuracy validation)  ← NEW v1.9

---

> **Remember:** A passing spec means nothing if the modeling is wrong.  
> **Enumerate reality first. Prove safety second.**

