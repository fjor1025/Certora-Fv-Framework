# CERTORA VERIFICATION MASTER GUIDE

> **The Complete Framework for Formal Verification of Smart Contracts**  
> **Version:** 3.2 (Adversarial Verification Loop + Optimization Pressure + Temporal Depth + Design Hostility)  
> **Use this guide to verify ANY Solidity contract from scratch**

---

## TABLE OF CONTENTS

1. [Framework Overview](#1-framework-overview)
   1.4. [Adversarial Verification Model](#14-adversarial-verification-model) ← **NEW v3.0**
2. [Project Setup](#2-project-setup)
3. [Phase 0: Contract Analysis](#3-phase-0-contract-analysis)
4. [Phase -1: Execution Closure](#4-phase--1-execution-closure)
5. [Phase 2: Property Discovery](#5-phase-2-property-discovery)
6. [Phase 2.5: Classification](#6-phase-25-classification)
7. [Phase 3.5: Causal Validation](#7-phase-35-causal-validation)
   7.6. [Multi-Epoch Attack Modeling](#76-multi-epoch-attack-modeling) ← **NEW v3.2**
8. [Phase 4-6: Modeling & Sanity](#8-phase-4-6-modeling--sanity)
   8.4. [Adversarial Design Interrogation](#84-adversarial-design-interrogation) ← **NEW v3.2**
9. [Phase 7: Write CVL](#9-phase-7-write-cvl)
9.5. [Phase 8: Attack Synthesis (Offensive)](#95-phase-8-attack-synthesis-offensive) ← **NEW v3.0**
   9.5.10. [Attacker Optimization & Profit Escalation](#9510-attacker-optimization--profit-escalation) ← **NEW v3.2**
10. [Running & Debugging](#10-running--debugging)
11. [Templates](#11-templates)
12. [Quick Reference](#12-quick-reference)
13. [Quick Start Chat Prompts](#13-quick-start-chat-prompts)

---

# 1. FRAMEWORK OVERVIEW

## 1.1 Your Framework Documents

| Document | Purpose | When to Use |
|----------|---------|-------------|
| **certora-master-guide.md** | Complete step-by-step instructions | Starting any new verification |
| **cvl-language-deep-dive.md** | Complete CVL language reference | ← **NEW in v1.5** |
| **verification-playbooks.md** | Worked examples (ERC-20, WETH, ERC-721) | ← **NEW in v1.5** |
| **advanced-cli-reference.md** | Performance optimization & advanced flags | ← **NEW in v1.4** |
| **impact-spec-template.md** | Anti-invariant & profit threshold rules | ← **NEW in v3.0** |
| **multi-step-attacks-template.md** | Flash loan, sandwich, staged, cross-contract patterns | ← **NEW in v3.0** |
| **offensive-pipeline.md** | CI/CD pipeline, CE triage, attack prioritization | ← **NEW in v3.0** |
| **poc-template-foundry.md** | CE → Foundry exploit PoC conversion | ← **ENHANCED in v3.0** |
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
│   UNDERSTAND → ENUMERATE → VALIDATE → ATTACK ⇄ DEFEND → PROVE   │
│                                                                  │
│   ❌ Never write CVL before completing phases 0 through 3.5     │
│   ❌ Never skip causal validation                                │
│   ❌ Never assume external contracts are "standard"              │
│   ❌ Never write a full defensive spec before an offensive       │
│      spec exists — they evolve together from the causal model   │
│   ❌ Never trust an offensive spec without a stated design intent│
│                                                                  │
│   "Causal validation defines reality. Defensive spec states     │
│    intent. Offensive spec attacks intent. The loop refines       │
│    both. Proof comes last."                                     │
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
│                              ┌─────────────────┐                     │
│                              │ SHARED CAUSAL    │                    │
│                              │ MODEL            │                    │
│                              └────────┬────────┘                     │
│                         ┌─────────────┴─────────────┐                │
│                         ▼                           ▼                │
│                ┌──────────────┐            ┌──────────────┐          │
│                │  DEFENSIVE   │◄──────────►│  OFFENSIVE   │          │
│                │  Hypothesis  │  FEEDBACK  │  Existential │          │
│                │  (minimal)   │    LOOP    │  Spec        │          │
│                └──────────────┘            └──────────────┘          │
│                         │                           │                │
│                         └─────────────┬─────────────┘                │
│                                       ▼                              │
│                            ┌────────────────┐                        │
│                            │ FINAL DEFENSIVE│                        │
│                            │ PROOF          │                        │
│                            │ (always last)  │                        │
│                            │   ✅ DONE      │                        │
│                            └────────────────┘                        │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

## 1.4 Adversarial Verification Model

> **Memorize this.** The framework is NOT linear (Causal → Defensive → Offensive).
> The framework is a **bidirectional feedback loop** where offensive and defensive
> specs evolve together on a shared causal model, and final defensive proof comes LAST.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ADVERSARIAL VERIFICATION FRAMEWORK                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                     ┌──────────────────────┐                                │
│                     │   CAUSAL VALIDATION   │                               │
│                     │   (Phases 0–6)        │                               │
│                     └──────────┬───────────┘                                │
│                                │                                             │
│                  Defines reachable state space                               │
│                                │                                             │
│                                ▼                                             │
│                     ┌──────────────────────┐                                │
│                     │   SHARED CAUSAL       │                               │
│                     │   MODEL               │ ← Neither spec is truth.      │
│                     └──────────┬───────────┘   The causal model is truth.   │
│                                │                                             │
│                ┌───────────────┴───────────────┐                            │
│                │                               │                            │
│                ▼                               ▼                            │
│   ┌────────────────────────┐     ┌────────────────────────┐                │
│   │  DEFENSIVE HYPOTHESIS  │     │  OFFENSIVE EXISTENTIAL │                │
│   │  (minimal intent)      │     │  SPEC (profit search)  │                │
│   │                        │     │                        │                │
│   │  "What must never      │     │  "Can an attacker      │                │
│   │   happen by design?"   │     │   extract profit?"     │                │
│   └────────────┬───────────┘     └────────────┬───────────┘                │
│                │                               │                            │
│                └───────────┬───────────────────┘                            │
│                            │                                                 │
│                            ▼                                                 │
│               ┌─────────────────────────┐                                   │
│               │   FEEDBACK LOOP         │                                   │
│               │                         │                                   │
│               │  SAT offensive →        │                                   │
│               │    exploit OR update    │                                   │
│               │    defensive hypothesis │                                   │
│               │                         │                                   │
│               │  UNSAT offensive →      │                                   │
│               │    weaken assumptions   │                                   │
│               │    OR expand causal     │                                   │
│               │    model                │                                   │
│               └─────────┬───────────────┘                                   │
│                         │                                                    │
│                    Loop until convergence                                    │
│                         │                                                    │
│                         ▼                                                    │
│               ┌─────────────────────────┐                                   │
│               │   FINAL DEFENSIVE       │                                   │
│               │   VERIFICATION          │                                   │
│               │   (ALWAYS LAST)         │                                   │
│               └─────────────────────────┘                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Canonical Loop

1. **Causal Validation** (Phases 0–6) establishes the reachable state space
2. **Adversarial Design Interrogation** (§8.4): Question the design itself before writing any spec
3. **Minimal Defensive Hypothesis**: State the design intent — what must never happen
4. **Offensive Existential Spec**: Search for economically profitable counterexamples
5. **Bidirectional Feedback**:
   - SAT offensive result → either a true exploit, or the defensive hypothesis needs updating
   - UNSAT offensive result → either the attack is impossible, or offensive assumptions need weakening
6. **Profit Escalation** (§9.5.10): Establish the maximum extractable value boundary, not just existence
7. **Multi-Epoch Analysis** (§7.6): Verify that attacks requiring state priming or delayed extraction are covered
8. **Loop** until both specs converge on the shared causal model
9. **Final Defensive Verification** — prove safety only after exhausting the attack surface

### Mutual Adversarial Hypothesis

> Defensive specifications and offensive specifications are **not sequential stages**.
> They are **mutually adversarial hypotheses** over the same causal model.
>
> * Defensive specifications express intended prohibitions.
> * Offensive specifications challenge whether those prohibitions meaningfully prevent economic impact.
>
> A defensive proof is considered **valid** only if:
> * Offensive specifications fail **without artificial assumptions**
> * Attacker profit cannot be escalated beyond the established boundary
> * Multi-epoch attacks are accounted for

### Vulnerability Definition

> A **vulnerability** is economically harmful behavior that is **allowed by design**
> (the spec permits it) or **reachable despite intended rules** (the spec is incomplete).

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

## 7.6 Multi-Epoch Attack Modeling

Many economically significant exploits do not occur in a single transaction or atomic execution.
Instead, they emerge through **state priming, distortion, and delayed extraction**.

The framework treats **multi-epoch attacks as first-class citizens**.

### Epoch Model

An attack may consist of multiple epochs:

| Epoch | Name | Purpose | Example |
|-------|------|---------|---------|
| **0** | **Setup** | Accumulate positions, approvals, or liquidity | Deposit large amount, get governance token |
| **1** | **Distortion** | Manipulate prices, ratios, or accounting state | Oracle manipulation, share inflation |
| **2** | **Extraction** | Realize profit via withdrawal, liquidation, or mint | Withdraw inflated shares, liquidate underwater positions |
| **3** | **Exit / Cleanup** | Reset state or externalize gains | Return flash loan, transfer profits out |

### Specification Guidance

Offensive specifications SHOULD:

1. **Explicitly allow state transitions across epochs** — use `persistent ghost` variables that survive across function calls and external interactions
2. **Avoid assumptions of atomicity** unless the protocol design enforces it (e.g., single-block flash loans)
3. **Treat delayed profit realization as valid impact** — attacker profit may only materialize in Epoch 2 or 3
4. **Model inter-block state** when relevant — some attacks require waiting for price updates, time locks, or oracle refreshes

### When Multi-Epoch Modeling Is Required

| Protocol Feature | Why Single-TX Is Insufficient |
|------------------|-------------------------------|
| Time-delayed withdrawals | Attacker primes state, waits, then extracts |
| Governance voting | Accumulate votes over time, execute at quorum |
| Interest accrual | Small manipulations compound over periods |
| Oracle price feeds | Distort price in block N, exploit in block N+1 |
| Liquidity pools | Front-run large trades across blocks |

### Rationale

> Restricting analysis to single-transaction models systematically excludes the
> most damaging business-logic vulnerabilities. Any attack that requires "wait
> and extract" is invisible to single-TX analysis.

**Reference:** [multi-step-attacks-template.md](multi-step-attacks-template.md) for implementation patterns.

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

## 8.4 Adversarial Design Interrogation

> **MANDATORY CHECKPOINT** — Before writing any defensive or offensive specification,
> the engineer must interrogate the *design itself* as a potential source of exploitability.

This step assumes:

> The code may be correct, and the design may still be extractable.

### Why This Step Exists

Business-logic vulnerabilities arise when designs are *internally consistent but externally exploitable*.
Formal verification that does not question design intent risks proving the safety of an extractive system.

Top-tier auditors ask: **"Who benefits from this design, and why?"**

### Mandatory Questions

At least one of the following MUST be explicitly answered before entering Phase 7:

| # | Question | What It Detects |
|---|----------|-----------------|
| 1 | Who benefits most if the protocol behaves *as specified*? | Privileged-actor extraction |
| 2 | What invariant would users *assume* exists but is not enforced? | Missing safety rails |
| 3 | Does the protocol reward adversarial strategy by design? | Incentive misalignment |
| 4 | Can "optimal behavior" be economically malicious? | Game-theoretic attack surfaces |
| 5 | What happens if all actors behave selfishly and rationally? | Tragedy-of-the-commons risks |

### Output

This step produces three artifacts that directly inform offensive specifications:

1. **Candidate attacker objectives** — what an attacker would optimize for
2. **Candidate profit metrics** — how to measure extraction (value delta, share dilution, fee capture)
3. **Candidate state variables of leverage** — which storage slots an attacker would manipulate

> Record these in `spec_authoring/design_interrogation.md` before proceeding.

### Example

**Protocol:** Vault with fee-on-transfer tokens

| Question | Answer |
|----------|--------|
| Who benefits as specified? | Vault owner (earns fees). But depositors assume 1:1 deposit-to-shares ratio. |
| Missing invariant? | No enforced relationship between `deposit amount` and `shares minted` after fee deduction. |
| Adversarial optimal strategy? | Deposit dust, inflate share price, front-run large deposits for dilution profit. |
| Design-level output → | Anti-invariant: `satisfy shares_minted_for_attacker * total_assets > attacker_deposit * total_shares` |

---

# 9. PHASE 7: WRITE CVL

> **You may only enter this phase after Phase 6 sanity gate PASSES**  
> **Begin with a MINIMAL defensive hypothesis — the full spec is written LAST,**  
> **after the adversarial verification loop (Section 1.4 / 9.5) converges.**

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

# 9.5. PHASE 8: ATTACK SYNTHESIS (OFFENSIVE)

> **v3.2 — Adversarial Verification Loop**  
> After Phase 6 sanity gate passes, the **shared causal model** is established.  
> From this model, offensive and defensive specs evolve together in a  
> **bidirectional feedback loop** — not in parallel, not in sequence.  
> Final defensive proof comes LAST, after the attack surface is exhausted.

## 9.5.1 The Adversarial Verification Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ADVERSARIAL VERIFICATION LOOP                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   After causal validation establishes the reachable state space,            │
│   the engineer formulates BOTH:                                             │
│                                                                              │
│     1. A MINIMAL DEFENSIVE HYPOTHESIS                                       │
│        What the design claims must never occur.                             │
│        (Intent, not full spec — just enough to state the claim.)            │
│                                                                              │
│     2. AN OFFENSIVE EXISTENTIAL SPECIFICATION                               │
│        Whether economically meaningful extraction is possible.              │
│        (Profit search, anti-invariants, attack patterns.)                   │
│                                                                              │
│   These specifications are refined in a FEEDBACK LOOP:                      │
│                                                                              │
│     ┌──────────────────────────────────────────────────────────────────┐    │
│     │                                                                  │    │
│     │  SAT offensive spec →                                            │    │
│     │    • If economically meaningful: EXPLOIT FOUND → fix code        │    │
│     │    • If not meaningful: defensive hypothesis was too weak →      │    │
│     │      UPDATE defensive hypothesis and loop                       │    │
│     │                                                                  │    │
│     │  UNSAT offensive spec →                                          │    │
│     │    • Attack is impossible under current model → good, BUT:      │    │
│     │    • Are offensive assumptions too strong? → WEAKEN and re-run  │    │
│     │    • Is the causal model incomplete? → EXPAND and re-validate   │    │
│     │                                                                  │    │
│     └──────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│   Only after offensive attack synthesis is EXHAUSTED is full                │
│   defensive correctness proven. Safety proofs come LAST, always last.       │
│                                                                              │
│   NEITHER SPEC IS TRUTH. THE CAUSAL MODEL IS TRUTH.                        │
│   Both specs are hypotheses tested against the shared causal model.         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Key Insight:** The old model said "prove correctness, then check for attacks" (linear)
> or "prove correctness while checking for attacks" (parallel). Both are wrong.
> The correct model: **offensive findings refine the defensive hypothesis, and
> defensive intent constrains the offensive search.** They are coupled, not independent.
>
> **Validity Criterion:** A defensive proof is only valid when:
> 1. Offensive specs fail without artificial assumptions
> 2. Profit escalation reaches an UNSAT boundary (§9.5.10)
> 3. Multi-epoch attack patterns are accounted for (§7.6)
> 4. Adversarial design interrogation has been completed (§8.4)

## 9.5.2 Economic Impact Baseline

For each asset in the protocol, establish:

| Asset | Total Value | Contract(s) Holding | Entry Points | Exit Points |
|-------|-------------|---------------------|--------------|-------------|
| ETH | $X | Vault, Escrow | deposit, receive | withdraw, transfer |
| Token | $Y | Staking, Treasury | stake, deposit | unstake, withdraw, redeem |
| Shares | N shares | Pool | mint | burn, redeem |

## 9.5.3 Attacker Objective Definition

Define what a successful attack looks like:

```cvl
// ═══════════════════════════════════════════════════════════════
// ATTACKER VALUE TRACKING
// ═══════════════════════════════════════════════════════════════

// Track value controlled by each address
persistent ghost mapping(address => mathint) actor_value {
    init_state axiom forall address a. actor_value[a] == 0;
}

// Track total system value
ghost mathint total_system_value {
    init_state axiom total_system_value == 0;
}

// Hook on balance changes to track value flows
hook Sstore balanceOf[KEY address account] uint256 newBal (uint256 oldBal) {
    mathint delta = to_mathint(newBal) - to_mathint(oldBal);
    actor_value[account] = actor_value[account] + delta;
}

// ═══════════════════════════════════════════════════════════════
// ATTACKER OBJECTIVE DEFINITIONS
// ═══════════════════════════════════════════════════════════════

// Attacker goal: extract value without providing equivalent input
definition profitable_attack(address attacker, mathint before, mathint after) 
    returns bool = after > before;

// System impact: protocol loses value
definition harmful_extraction(mathint sys_before, mathint sys_after) 
    returns bool = sys_after < sys_before;
```

## 9.5.4 Anti-Invariant Construction

Write rules that **SHOULD FAIL**. If they pass, no attack found. If they fail, you've discovered an exploit:

```cvl
/**
 * @title Attacker Profit Search
 * @notice Searches for any function that allows caller to profit
 * @dev EXPECTED TO FAIL if exploit exists
 * 
 * If this rule is VIOLATED:
 *   - The counterexample contains the attack parameters
 *   - The function called is the attack vector
 *   - The profit amount is the exploit impact
 */
rule attacker_cannot_profit(env e, method f) 
    filtered { f -> !f.isView && !f.isFallback }
{
    require e.msg.sender != 0;
    require e.msg.value == 0;  // Adjust for payable functions
    
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
 * @notice Verifies protocol value doesn't leak
 * @dev EXPECTED TO FAIL if value can be extracted
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
        "EXPLOIT FOUND: System value changed unexpectedly";
}
```

## 9.5.5 Attack Search Process

### Step 1: Run Anti-Invariants

```bash
# Run profit search across all state-changing functions
certoraRun config.conf --rule attacker_cannot_profit

# Run system value conservation check
certoraRun config.conf --rule system_value_conserved
```

### Step 2: Interpret Results

| Result | Meaning | Action |
|--------|---------|--------|
| **VERIFIED** | No profit path found for this rule | Good — or model incomplete |
| **VIOLATED** | Attack path found | ⚠️ EXPLOIT — examine CE immediately |
| **TIMEOUT** | Search space too large | Add constraints, simplify model |

### Step 3: Use `satisfy` for Active Attack Search

```cvl
/**
 * @title Find Profitable Inputs
 * @notice Actively searches for inputs that create profit
 * @dev If this VERIFIES, the witness contains exploit parameters
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
    
    // FIND inputs that create profit > threshold
    // If VERIFIED, the witness contains attack parameters
    satisfy profit > 1000000;  // Profit > 1M units
}
```

## 9.5.6 Multi-Step Attack Patterns

Real exploits are rarely single-transaction. Use these patterns:

### Flash Loan Pattern

```cvl
rule flash_loan_attack_search(env e1, env e2) {
    address attacker = e1.msg.sender;
    require e2.msg.sender == attacker;
    require e1.block.number == e2.block.number;  // Same block (atomic)
    
    mathint initial = actor_value[attacker];
    
    // Step 1: Borrow + manipulate
    calldataarg args1;
    method f1;
    f1(e1, args1);
    
    // Step 2: Extract + repay
    calldataarg args2;
    method f2;
    f2(e2, args2);
    
    mathint final_val = actor_value[attacker];
    
    // Should not profit from flash loan sequence
    assert final_val <= initial,
        "FLASH LOAN EXPLOIT: Attacker profited";
}
```

### Sandwich Pattern

```cvl
rule sandwich_attack_search(env e_front, env e_victim, env e_back) {
    address attacker = e_front.msg.sender;
    address victim = e_victim.msg.sender;
    
    require e_back.msg.sender == attacker;
    require attacker != victim;
    
    // Same block
    require e_front.block.number == e_victim.block.number;
    require e_victim.block.number == e_back.block.number;
    
    mathint attacker_initial = actor_value[attacker];
    mathint victim_initial = actor_value[victim];
    
    // Frontrun → Victim → Backrun
    method f; calldataarg args1; calldataarg args2; calldataarg args3;
    f(e_front, args1);
    f(e_victim, args2);
    f(e_back, args3);
    
    mathint attacker_final = actor_value[attacker];
    mathint victim_final = actor_value[victim];
    
    mathint attacker_profit = attacker_final - attacker_initial;
    
    assert attacker_profit <= 0,
        "SANDWICH EXPLOIT: Attacker profited from ordering";
}
```

## 9.5.7 Counterexample → Exploit Conversion

When an anti-invariant fails, convert the CE to a Foundry PoC:

### Extraction Template

```markdown
## Exploit Report: [Rule Name]

**Date:** [YYYY-MM-DD]
**Rule:** `attacker_cannot_profit`
**Severity:** [CRITICAL/HIGH/MEDIUM]

### Attack Parameters (from CE)

| Parameter | Value |
|-----------|-------|
| Function | `withdraw(uint256)` |
| Caller | `0xAttacker` |
| Argument | `999999999999999999` |
| Pre-state balance | `100` |
| Post-state balance | `1000000000000000099` |
| Profit | `999999999999999999` |

### Root Cause

[Explain why this is possible — e.g., missing check, arithmetic error]

### Foundry PoC

```solidity
function test_exploit() public {
    // Setup: attacker has initial balance
    deal(address(token), attacker, 100);
    
    // Attack: call function with CE parameters
    vm.prank(attacker);
    vault.withdraw(999999999999999999);
    
    // Verify profit
    assertGt(token.balanceOf(attacker), 100);
}
```
```

## 9.5.8 Integration: The Bidirectional Loop in Practice

Offensive and defensive verification evolve together from the shared causal model.
This is NOT a parallel execution — it is a **feedback loop** where each informs the other.

> Defensive specifications express intended prohibitions.
> Offensive specifications challenge whether those prohibitions meaningfully prevent economic impact.
> **Neither is trusted. The causal model is truth. Both are hypotheses tested against it.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BIDIRECTIONAL VERIFICATION LOOP                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Phases 0-6: Analysis & Modeling → SHARED CAUSAL MODEL                     │
│                                                                              │
│   From the shared causal model, formulate BOTH:                             │
│                                                                              │
│   ┌────────────────────────────┐     ┌─────────────────────────────────┐    │
│   │  MINIMAL DEFENSIVE         │     │  OFFENSIVE EXISTENTIAL          │    │
│   │  HYPOTHESIS                │     │  SPECIFICATION                  │    │
│   │  ├── State design intent   │     │  ├── Import impact ghosts       │    │
│   │  ├── What must never happen│     │  ├── Write anti-invariants      │    │
│   │  └── Minimal, not full spec│     │  ├── Run profit search rules    │    │
│   └─────────────┬──────────────┘     │  ├── Run multi-step attacks     │    │
│                 │                     │  └── Check hook liveness first  │    │
│                 │                     └──────────────┬──────────────────┘    │
│                 │                                    │                       │
│                 └──────────────┬──────────────────────┘                      │
│                                │                                             │
│                                ▼                                             │
│                     ┌──────────────────┐                                    │
│                     │  FEEDBACK LOOP:  │                                    │
│                     │  SAT → exploit   │                                    │
│                     │    or update     │                                    │
│                     │    hypothesis    │                                    │
│                     │  UNSAT → weaken  │                                    │
│                     │    assumptions   │                                    │
│                     └────────┬─────────┘                                    │
│                              │                                               │
│                         Converged?                                           │
│                              │                                               │
│                              ▼                                               │
│                     ┌──────────────────┐                                    │
│                     │ FINAL DEFENSIVE  │                                    │
│                     │ VERIFICATION     │ ← Full spec now, not before        │
│                     │ (ALWAYS LAST)    │                                    │
│                     └──────────────────┘                                    │
│                                                                              │
│   Complete: Attack surface exhausted AND defensive proof holds              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 9.5.9 Phase 8 Checklist

Before declaring verification complete:

- [ ] Imported impact tracking ghosts from `impact-spec-template.md` (ALL must be `persistent`)
- [ ] Adapted value flow hooks to protocol's storage layout
- [ ] Completed Value Flow Completeness Checklist (all assets covered)
- [ ] Ran `impact_hook_liveness` for ALL state-changing functions ← **NEW**
- [ ] Ran `system_value_hook_liveness` for value-moving functions ← **NEW**
- [ ] Fixed any hooks where liveness `satisfy` was VIOLATED ← **NEW**
- [ ] Ran `attacker_cannot_profit` across all functions
- [ ] Ran `system_value_conserved` 
- [ ] Ran `find_profitable_inputs` satisfy rule
- [ ] Ran `find_max_profit_threshold` with iterative tightening ← **NEW**
- [ ] Applied flash loan pattern (if protocol holds value)
- [ ] Applied sandwich pattern (if protocol has price-sensitive ops)
- [ ] Applied cross-contract pattern (if protocol interacts with external contracts) ← **NEW**
- [ ] Converted any CEs to Foundry PoCs
- [ ] All PoCs reproduce the exploit
- [ ] All exploits fixed and re-verified
- [ ] Re-ran defensive verification after fixes

**Reference:** 
- [impact-spec-template.md](impact-spec-template.md) — Complete economic impact tracking infrastructure
- [multi-step-attacks-template.md](multi-step-attacks-template.md) — Flash loan, sandwich, staged attack patterns
- [offensive-pipeline.md](offensive-pipeline.md) — Sample `.conf`, CI pipeline, CE severity triage, attack prioritization

## 9.5.10 Attacker Optimization & Profit Escalation

Offensive specifications must not stop at proving the *existence* of a profitable execution.
In adversarial environments, the attacker's objective is to **maximize extractable value**, not merely demonstrate feasibility.

Therefore, offensive verification must include **profit escalation pressure**.

### Definition

An **optimized attack** is an execution trace such that:

* Attacker profit is maximized under the causal model
* No strictly greater profit trace exists under equivalent assumptions

### Required Practice

Offensive specifications SHOULD:

1. **Parameterize attacker profit thresholds** (e.g., `≥ X`)
2. **Iteratively increase thresholds** until the Prover returns UNSAT
3. **Detect unbounded or monotonic profit growth** — if every threshold is SAT, the vulnerability is unbounded
4. **Record the SAT→UNSAT boundary** — this is the maximum extractable value under the causal model

### Escalation Protocol

```
┌─────────────────────────────────────────────────────────────────┐
│  PROFIT ESCALATION PROTOCOL                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Run:  satisfy attacker_profit >= 1                          │
│     SAT?  → Attack exists. Record witness.                      │
│                                                                  │
│  2. Run:  satisfy attacker_profit >= 10^3                       │
│     SAT?  → Escalate.                                           │
│                                                                  │
│  3. Run:  satisfy attacker_profit >= 10^6                       │
│     SAT?  → Escalate.                                           │
│                                                                  │
│  4. Continue: 10^9, 10^12, 10^15, 10^18                        │
│     First UNSAT = maximum extractable value boundary.           │
│                                                                  │
│  5. If ALL thresholds SAT → UNBOUNDED VULNERABILITY             │
│     Report as CRITICAL. Immediate mitigation required.          │
│                                                                  │
│  Note: The SAT→UNSAT boundary defines the "dominant exploit"   │
│  — the attack an attacker would actually execute in practice.   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Rationale

A specification that proves `profit ≥ 1` while missing `profit ≥ 100` is **not sufficient**
for real-world security conclusions — whether in internal security reviews, private audits, bug bounties, or production deployments.

* Existence-only proofs detect *vulnerabilities*
* Optimization-driven proofs detect *dominant exploits*

> **Rule:** No offensive verification is complete until the profit boundary is established
> or the Prover proves no profitable execution exists at any threshold.

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

□ ════════════════════════════════════════════════════════════════════════════
□ ADVERSARIAL DESIGN INTERROGATION — BEFORE any spec writing             ← NEW v3.2
□ ════════════════════════════════════════════════════════════════════════════
□ §8.4: Design interrogation complete (5 mandatory questions answered)
□ §8.4: Candidate attacker objectives documented
□ §8.4: Candidate profit metrics defined
□ §8.4: Candidate state variables of leverage identified

□ ════════════════════════════════════════════════════════════════════════════
□ SHARED CAUSAL MODEL ESTABLISHED
□ ════════════════════════════════════════════════════════════════════════════
□ Minimal defensive hypothesis stated (design intent — what must never happen)
□ Offensive existential spec written (impact ghosts, anti-invariants, profit search)

□ ════════════════════════════════════════════════════════════════════════════
□ OFFENSIVE ATTACK SYNTHESIS — exhausted BEFORE final defensive proof
□ ════════════════════════════════════════════════════════════════════════════
□ Phase 8: Economic impact baseline documented (assets, values, entry/exit)  ← NEW v3.0
□ Phase 8: Impact ghosts created (actor_value, total_system_value) — ALL persistent  ← UPDATED
□ Phase 8: Value Flow Completeness Checklist completed                       ← NEW
□ Phase 8: Hook liveness checked (impact_hook_liveness for all functions)    ← NEW
□ Phase 8: Anti-invariants written (rules expected to FAIL)                  ← NEW v3.0
□ Phase 8: Profit search rules executed (satisfy-based attack search)        ← NEW v3.0
□ Phase 8: Profit escalation protocol completed (§9.5.10)                     ← NEW v3.2
□ Phase 8: SAT→UNSAT boundary established (max extractable value)            ← NEW v3.2
□ Phase 8: Multi-epoch attack patterns verified (§7.6)                        ← NEW v3.2
□ Phase 8: Iterative threshold protocol run (find_max_profit_threshold)      ← NEW
□ Phase 8: Multi-step attacks tested (flash loan, sandwich, staged, cross-contract) ← UPDATED
□ Phase 8: Any failing anti-invariant → CE converted to Foundry PoC          ← NEW v3.0
□ Phase 8: All anti-invariants PASS (no profitable attack found)             ← NEW v3.0
□ Phase 8: Feedback loop converged — offensive SAT/UNSAT results reviewed

□ ════════════════════════════════════════════════════════════════════════════
□ FINAL DEFENSIVE VERIFICATION — ALWAYS LAST
□ ════════════════════════════════════════════════════════════════════════════
□ Phase 7: Full CVL spec written (only after feedback loop converges)
□ Prover: All defensive rules PASS
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
>  
> **Conceptual Model (Section 1.4):** These prompts encode the **Adversarial Verification  
> Loop** — after causal validation, offensive and defensive specs evolve together on a  
> shared causal model. Offensive attack synthesis is exhausted BEFORE final defensive  
> proof. Neither spec is truth; the causal model is truth.

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

**Key references to use throughout (v3.2):**
- cvl-language-deep-dive.md — Complete CVL language reference (types, ghosts, hooks, operators, builtin rules §19.1)
- verification-playbooks.md — Worked examples for ERC-20, WETH, ERC-721 + Phase 0 builtin scan
- best-practices-from-certora.md — Sections 7-9 (vacuity defense, requireInvariant lifecycle, edge cases)
- best-practices-from-certora.md — §8.4 Circular dependency detection for invariant DAGs  ← NEW v1.9
- certora-spec-framework.md — Revert/failure-path patterns (Pattern B: @withrevert PREFERRED)
- certora-spec-framework.md — Custom summary accuracy protocol & invariant DAG protocol  ← NEW v1.9
- categorizing-properties.md — MUST REVERT WHEN checklist for property discovery
- impact-spec-template.md — Economic impact tracking (persistent ghosts, anti-invariants, hook liveness)  ← NEW v3.0
- multi-step-attacks-template.md — Flash loan, sandwich, staged, cross-contract attack patterns  ← NEW v3.0
- offensive-pipeline.md — Sample .conf files, CI pipeline script, CE severity triage, attack prioritization  ← NEW v3.0
- categorizing-properties.md §0 — Economic Impact Categories & Attacker Objective Checklist  ← NEW v3.0
- certora-master-guide.md §8.4 — Adversarial Design Interrogation (MANDATORY before any spec)  ← NEW v3.2
- certora-master-guide.md §7.6 — Multi-Epoch Attack Modeling (time-delayed/oracle protocols)  ← NEW v3.2
- certora-master-guide.md §9.5.10 — Profit Escalation Protocol (SAT→UNSAT boundary)          ← NEW v3.2
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
- **Economic Impact Categories** — categorizing-properties.md §0 (Value Extraction, Insolvency,  ← NEW v3.0
  Share Dilution, Debt Socialization, Liquidity Freeze, Governance Capture, Front-Running, Griefing)
- Valid States (range constraints)
- State Transitions (function effects)
- System-Level (aggregates, sums)
- Access Control (who can do what)
- **Revert Behavior (MUST REVERT WHEN)**  ← NEW v1.6

**Apply these techniques:**
- **Attacker Objective Checklist** — categorizing-properties.md §0 (what can an attacker profit from?)  ← NEW v3.0
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
   - Economic Impact / Valid States / State Transitions / System-Level / Threat-Driven / Revert Behavior

**4. Economic Impact Discovery (v3.2):**
   - For each asset: Can an attacker extract value? Cause insolvency? Dilute shares?
   - Use categorizing-properties.md §0 Attacker Objective Checklist
   - Document findings in candidate_properties.md under new "Impact" category

Reference:
- categorizing-properties.md sections 5 and 6 (Dual Checklist & Test Mining)
- categorizing-properties.md §0 Economic Impact Categories & Attacker Objective Checklist  ← NEW v3.0
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

## 13.5 For Phase 7 (Validation PASSED → Minimal Defensive Hypothesis + Offensive Spec)

```markdown
My validation spec PASSED for [ContractName]. Ready to begin the adversarial
verification loop (see Section 1.4 — Adversarial Verification Model).

**Target:** [path/to/ContractName.sol]
**Token Standard (if any):** [ERC-20 / ERC-721 / WETH / None]
**Validation Spec:** certora/specs/validation_{target}.spec (PASSED ✅)
**Validation Evidence Review:** COMPLETED ✅  ← NEW v2.0
**Design Interrogation (§8.4):** COMPLETED ✅  ← NEW v3.2
**Candidate Properties:** spec_authoring/{target}_candidate_properties.md
**Attacker Objectives (from §8.4):** [list from design interrogation]              ← NEW v3.2
**Profit Metrics (from §8.4):** [value delta / share dilution / fee capture / etc] ← NEW v3.2

**⚠️ IMPORTANT: This is NOT "write full defensive spec, then check for attacks."**
**This is a BIDIRECTIONAL FEEDBACK LOOP — offensive and defensive evolve together.**

Please help me set up BOTH specs from the shared causal model:

**A. Minimal Defensive Hypothesis (intent, not full spec):**
1. Copy infrastructure from validation spec (methods, ghosts, hooks)
2. DELETE all validation_* rules
3. State the MINIMAL design intent — what the protocol claims must never happen
4. Write only the core safety invariants from candidate_properties.md
5. Use `requireInvariant` (not raw `require`) for proven invariant preconditions
6. **Do NOT write the full spec yet** — it will be refined by offensive findings

**B. Offensive Existential Spec (attack search):**
7. Set up impact ghosts in a separate impact spec file (actor_value, total_system_value)
8. Write anti-invariants: rules expected to FAIL if exploit exists
9. Run hook liveness checks FIRST (dead hooks = vacuous results)
10. Run profit search rules (`satisfy attacker_profit >= 1`)
11. **Run Profit Escalation Protocol (§9.5.10):** iterate thresholds (1, 10^3, 10^6, 10^9, 10^12, 10^15, 10^18) until UNSAT — establish the SAT→UNSAT boundary (max extractable value)
12. **Check multi-epoch attack patterns (§7.6)** if the protocol has time-delayed operations, interest accrual, or oracle feeds

**C. The Feedback Loop — iterate until convergence:**
13. If offensive SAT (exploit found) → fix code OR update defensive hypothesis → re-run
14. If offensive UNSAT → weaken offensive assumptions OR expand causal model → re-run
15. Loop until both specs converge on the shared causal model

**D. Final Defensive Verification (ALWAYS LAST):**
16. Only after offensive attack surface is EXHAUSTED:
17. Complete the full CVL spec with all invariants, rules, and @withrevert patterns
18. Add standard `definition` blocks (nonpayable, nonzerosender, balanceLimited)
19. For EVERY state-changing function, write @withrevert rules (Pattern B: biconditional <=>)
20. Add `use builtin rule uncheckedOverflow;` and/or `safeCasting;` if applicable
21. **Annotate every invariant with `@dev Level: N | Dependencies: ...`**  ← NEW v1.9
22. **For each custom summary, add accuracy annotation (Exact/Over/Under)**  ← NEW v1.9
23. Create final certora/specs/{Contract}.spec and certora/confs/{Contract}.conf

Reference:
- certora-master-guide.md Section 1.4 (Adversarial Verification Model — the canonical loop)
- certora-master-guide.md section 9.0 (Transition from Validation to Real Spec)
- certora-master-guide.md section 9.5 (Phase 8: Attack Synthesis)
- certora-master-guide.md section 9.5.10 (Attacker Optimization & Profit Escalation)        ← NEW v3.2
- certora-master-guide.md section 7.6 (Multi-Epoch Attack Modeling)                          ← NEW v3.2
- cvl-language-deep-dive.md (complete CVL reference — types, operators, ghosts, hooks, definitions)
- cvl-language-deep-dive.md §19.1 (builtin rules — uncheckedOverflow, safeCasting)  ← NEW v1.7
- verification-playbooks.md (if ERC-20/721/WETH — follow the complete worked example)
- certora-spec-framework.md (CVL syntax patterns + Revert/Failure-Path Checklist)  ← updated v1.6
- impact-spec-template.md (economic impact tracking — persistent ghosts, anti-invariants)  ← NEW v3.0
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
6. **If this is an anti-invariant CE (Phase 8), convert to exploit:**  ← NEW v3.0
   - Extract function, caller, args, state from CE
   - Use poc-template-foundry.md to generate Foundry PoC
   - Validate on mainnet fork

Reference:
- certora-ce-diagnosis-framework.md (comprehensive 5-phase diagnosis + ghost havocing guide)
- certora-ce-diagnosis-framework.md SILENT PASS classification  ← NEW v1.6
- certora-ce-diagnosis-framework.md CE→Exploit Conversion section  ← NEW v3.0
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

## 13.7 For Phase 8 (Offensive Attack Synthesis — Bidirectional Loop) ← NEW v3.0

```markdown
Phase 6 sanity gate PASSED for [ContractName]. The shared causal model is
established. I'm entering the adversarial verification loop (Section 1.4).

**Target:** [path/to/ContractName.sol]
**Phase 6 Sanity Gate:** PASSED ✅
**Adversarial Design Interrogation (§8.4):** COMPLETED ✅                        ← NEW v3.2
**Phase:** Adversarial Verification Loop (offensive ⇄ defensive feedback)
**Minimal Defensive Hypothesis:** [stated / not yet stated]
**Protocol Type:** [AMM / Lending / Staking / Governance / Vault / Other]
**Assets at Risk:** [ETH, ERC-20 tokens, shares, LP tokens, etc.]
**External Integrations:** [oracles, other protocols, flash loan providers, etc.]

**⚠️ This is NOT a parallel run. This is a FEEDBACK LOOP.**
**Offensive findings refine the defensive hypothesis.**
**Defensive intent constrains the offensive search.**
**Final defensive proof comes LAST, after the loop converges.**

Please help me run the adversarial verification loop:

1. **State the Minimal Defensive Hypothesis:**
   - What does the design claim must never happen?
   - Write ONLY the core safety invariants (not the full spec)
   - This hypothesis will be refined by offensive findings

2. **Economic Impact Assessment:**
   - What assets are at risk? (ETH, tokens, shares, etc.)
   - What is the total value at stake?
   - What would a successful exploit look like?
   - Complete the Phase 0 Asset Flow Trace cross-reference

3. **Create Attack Search Spec (impact.spec):**
   - Copy ghosts from impact-spec-template.md
   - ⚠️ ALL ghosts MUST be `persistent` (survives HAVOC from external calls)
   - Set up `actor_value` persistent ghost to track value per address
   - Set up `total_system_value` persistent ghost for protocol health
   - Hook on ALL value-bearing state changes (not just `_balances`)
   - Complete the **Value Flow Completeness Checklist** — cross-reference every
     asset from Phase 0 against hooks (ERC-20, ETH, shares, LP, rebasing, fees)

4. **Run Hook Liveness Checks FIRST:**
   - Run `impact_hook_liveness` for every state-changing function
   - Run `system_value_hook_liveness` for value-moving functions
   - If any `satisfy` is VIOLATED → hooks are blind to that function's effects
   - Fix hooks BEFORE trusting any anti-invariant result
   - A passing anti-invariant with dead hooks proves NOTHING

5. **Write Anti-Invariants:**
   - `attacker_cannot_profit` — fails if caller can profit
   - `system_value_conserved` — fails if value leaks
   - `zero_sum_transfers` — fails if value created from nothing
   - Use `satisfy` with `find_profitable_inputs` for active attack search

6. **Run Iterative Threshold Protocol (find maximum exploit size — §9.5.10):**
   - `satisfy attacker_profit >= 1` → SAT? → Attack exists. Record witness. Continue.
   - `satisfy attacker_profit >= 10^3` → SAT? Continue
   - `satisfy attacker_profit >= 10^6` → SAT? Continue
   - `satisfy attacker_profit >= 10^9` → SAT? Continue
   - `satisfy attacker_profit >= 10^12` → SAT? Continue
   - Continue: `10^15`, `10^18` → First UNSAT = maximum extractable value boundary
   - If ALL thresholds SAT → **UNBOUNDED VULNERABILITY** — report as CRITICAL
   - Certora's SMT solver finds ANY witness, not the maximum — this iterative
     tightening protocol provides the workaround (see §9.5.10 for full protocol)

7. **Multi-Step Attack Patterns (choose by protocol type):**
   - Flash loan attack search (if protocol holds value)
   - Sandwich attack search (if protocol has price-sensitive operations)
   - Staged attack accumulation (if protocol has time-dependent state)
   - Governance attack patterns (if protocol has governance)
   - **Cross-contract attack search** (if protocol interacts with external contracts)
   - Use `filtered` clauses for multi-step rules to avoid TIMEOUT
   - See combinatorial explosion guidance in multi-step-attacks-template.md

8. **Feedback Loop — iterate until convergence:**
   - SAT offensive result → true exploit? Fix code. Not meaningful? Update
     defensive hypothesis and loop.
   - UNSAT offensive result → are assumptions too strong? Weaken and re-run.
     Is causal model incomplete? Expand and re-validate.
   - Loop until BOTH specs converge on the shared causal model.

9. **CE → Exploit Conversion (when offensive SAT is meaningful):**
   - Extract attack parameters from CE
   - Convert to Foundry PoC using poc-template-foundry.md
   - Validate exploit on mainnet fork
   - If exploit confirmed → fix code → re-run BOTH specs from the loop

10. **Final Defensive Verification (LAST — only after loop converges):**
    - Now write the FULL defensive spec (refined by all offensive findings)
    - Prove all safety invariants hold
    - This is the terminal step — proof comes last.

References:
- certora-master-guide.md Section 1.4 (Adversarial Verification Model — the canonical loop)
- certora-master-guide.md Section 7.6 (Multi-Epoch Attack Modeling)                         ← NEW v3.2
- certora-master-guide.md Section 8.4 (Adversarial Design Interrogation)                    ← NEW v3.2
- certora-master-guide.md Section 9.5 (Phase 8: Attack Synthesis)
- certora-master-guide.md Section 9.5.10 (Attacker Optimization & Profit Escalation)        ← NEW v3.2
- impact-spec-template.md (value tracking infrastructure + hook liveness + completeness checklist)
- multi-step-attacks-template.md (attack pattern library + cross-contract + depth guidance + multi-epoch)
- offensive-pipeline.md (sample .conf, CI pipeline, CE severity triage, attack prioritization)
- poc-template-foundry.md (CE → executable PoC)
- categorizing-properties.md §0 (Economic Impact Categories + Profit Escalation Categories)
```

---

## 13.8 Essential Information to Provide

When starting any verification conversation, always include:

| Required Info | Example |
|---------------|---------|
| **Project path** | `/home/user/my-protocol` |
| **Target contract** | `contracts/core/Vault.sol` |
| **Contract name** | `Vault` (as declared in Solidity) |
| **Dependencies** | `imports Token.sol, Oracle.sol, Utils.sol` |
| **Current phase** | Phase 0 / -1 / 2 / 2.5 / 3.5 / Adversarial Loop (offense ⇄ defense) / Final Proof |
| **Token standard** | ERC-20 / ERC-721 / WETH / None |
| **Protocol type** | AMM / Lending / Staking / Governance / Vault / Other ← NEW v3.0 |
| **Prover version** | v8.8.0+ / older ← NEW v1.7 |
| **unchecked{}/assembly?** | Yes / No ← NEW v1.7 |
| **Invariant dependencies?** | Complex chain / Simple / None ← NEW v1.9 |
| **Custom summaries?** | Yes / No ← NEW v1.9 |
| **External integrations?** | Oracles / Flash loans / Other protocols / None ← NEW v3.0 |

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
- Protocol type (AMM/Lending/Staking/Governance/Vault) for Phase 8 pattern selection  ← NEW v3.0
- External protocol integrations (oracles, flash loan providers, other DeFi)  ← NEW v3.0
- All asset types held by the protocol (for Value Flow Completeness Checklist)  ← NEW v3.0

---

> **Remember:** A passing spec means nothing if the modeling is wrong.  
> **Causal validation defines reality. Defensive spec states intent. Offensive spec attacks intent.**  
> **The loop refines both. Proof comes last.**

