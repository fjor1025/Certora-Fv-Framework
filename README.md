# Certora Formal Verification Framework

> **A complete, reusable framework for formal verification of Solidity smart contracts using Certora Prover**  
> **Version:** 1.5 (RareSkills Integration)

---

## What's New in v1.5

**RareSkills Integration** (sourced from the complete RareSkills Certora Book — 35 chapters, 60,000+ words):

**NEW Documents:**
- **CVL_LANGUAGE_DEEP_DIVE.md** — Complete CVL language reference covering 20 topics:
  - Type system (`mathint`, `require_uint256`, casting dangers)
  - Core statements (`require`, `assert`, `satisfy` with existential semantics)
  - Logical operators (`=>` implication, `<=>` biconditional, contrapositive)
  - Vacuous truth and tautology defenses
  - Method tags (`@withrevert`, `@norevert`, `lastReverted`)
  - Environment variables (`env`, `msg.sender`, `msg.value`, `nativeBalances`)
  - Ghost variables (initialization, havocing, persistent ghosts)
  - Hooks (Sstore delta, Sload constraint, CALL opcode)
  - `definition` blocks (reusable CVL expressions)
  - Invariants (base case, inductive step, preserved blocks)
  - `requireInvariant` lifecycle
  - Parametric and partially parametric rules
  - Liveness / Effect / No-Side-Effect pattern
  - DISPATCHER and external callback resolution
  - Loops, self-transfer handling, sanity checks

- **VERIFICATION_PLAYBOOKS.md** — Complete worked verification examples:
  - **ERC-20 Playbook** — 22 rules across 4 phases (Correctness, Side Effects, Invariants, Authorization)
  - **WETH Playbook** — Solvency invariant, deposit/withdraw, persistent ghost + CALL hook
  - **ERC-721 Playbook** — Mint/burn/transfer, helperSoundFnCall, DISPATCHER for callbacks

**Updated Documents:**
- `CERTORA_SPEC_FRAMEWORK.md` — Added `definition` blocks template, Liveness/Effect/No-Side-Effect rule template
- `BEST_PRACTICES_FROM_CERTORA.md` — Added vacuous truth defense, `require` → `requireInvariant` lifecycle, self-transfer edge cases
- `CERTORA_CE_DIAGNOSIS_FRAMEWORK.md` — Added ghost havocing diagnosis guide, persistent ghost patterns
- `CERTORA_MASTER_GUIDE.md` — Added references to new documents, version bump

**Why This Matters:**
This release integrates deep CVL language knowledge from the authoritative RareSkills Certora Book. Every gap identified in prior versions is now filled — from foundational type system semantics to production OpenZeppelin verification patterns.

---

## Previous Enhancements

**v1.4:** Performance Optimization + Advanced CLI Reference  
**v1.3:** Property Prioritization + Tutorial Best Practices + CE Investigation  
**v1.2:** Dual Mindset ("Should Always" / "Should Never") + Test Mining  
**v1.1:** Validation-to-Real-Spec Transition + Chat Prompts

---

## Framework Files

| File | Purpose | When to Use |
|------|---------|-------------|
| **INDEX.md** | Navigation guide & quick access | ← **START HERE for navigation** |
| **CERTORA_MASTER_GUIDE.md** | Complete step-by-step instructions | ← **START HERE for verification** |
| **CVL_LANGUAGE_DEEP_DIVE.md** | Complete CVL language reference | ← **NEW in v1.5** - CVL mastery |
| **VERIFICATION_PLAYBOOKS.md** | Worked examples (ERC-20, WETH, ERC-721) | ← **NEW in v1.5** - Copy & adapt |
| **ADVANCED_CLI_REFERENCE.md** | Performance & advanced flags | ← **NEW in v1.4** - Timeouts/optimization |
| **CERTORA_WORKFLOW.md** | Phase overview & checklist | Quick reference |
| **CERTORA_SPEC_FRAMEWORK.md** | CVL 2.0 syntax & templates | Writing actual CVL |
| **CERTORA_CE_DIAGNOSIS_FRAMEWORK.md** | Counterexample debugging | When rules fail |
| **SPEC AUTHORING (CERTORA).md** | Deep methodology & theory | Understanding WHY |
| **Categorizing_Properties.md** | Property discovery guidance | Phase 2 |
| **BEST_PRACTICES_FROM_CERTORA.md** | Official tutorial techniques | Property discovery & patterns |
| **QUICK_REFERENCE_v1.3.md** | Printable cheat sheet | Keep open while coding |
| **VERSION_HISTORY.md** | Version tracking & migration | Check what changed |

---

## Quick Start

### 1. Copy Framework to Your Project

```bash
# Copy all framework files to your project root
cp Certora-Fv-Framework/*.md /path/to/your-project/
```

### 2. Create Verification Structure

```bash
cd /path/to/your-project

# Set your target contract name
TARGET_CONTRACT="YourContract"
TARGET_LOWER="yourcontract"

# Create directories
mkdir -p spec_authoring
mkdir -p certora/{specs,confs,harnesses,helpers}

# Create analysis documents
touch "spec_authoring/${TARGET_LOWER}_spec_authoring.md"
touch "spec_authoring/${TARGET_LOWER}_candidate_properties.md"
touch "spec_authoring/${TARGET_LOWER}_causal_validation.md"

# Create spec files
touch "certora/specs/validation_${TARGET_LOWER}.spec"
touch "certora/specs/${TARGET_CONTRACT}.spec"
touch "certora/confs/validation_${TARGET_LOWER}.conf"
touch "certora/confs/${TARGET_CONTRACT}.conf"
```

### 3. Follow the Workflow

```
Phase 0   → Contract Analysis (entry points, storage, external calls)
Phase -1  → Execution Closure (external contracts, modeling decisions)
Phase 2   → Property Discovery (DUAL MINDSET: should always + should never)
            └── Mine tests for invariants, threats, blind spots
Phase 2.5 → Classification (INVARIANT vs RULE)
Phase 3.5 → Causal Validation ← RUN VALIDATION SPEC FIRST
Phase 4-6 → Modeling & Sanity Gate
Phase 7   → Write Real CVL Spec
```

---

## Dual Mindset Property Discovery (NEW in v1.2)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   VALIDATION MINDSET              ATTACKER MINDSET                      │
│   (What SHOULD happen)            (What MUST NEVER happen)              │
│                                                                          │
│   "This should always..."         "This should never..."                │
│   "When X, then Y"                "Even if X, never Y"                  │
│                                                                          │
│   + TEST MINING: Extract invariants, threats, and blind spots           │
│     from existing test suites                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

See `Categorizing_Properties.md` sections 5 and 6 for details.

---

## Project Structure After Setup

```
your-project/
│
├── ═══ FRAMEWORK FILES ═══════════════════════════════════════════
├── CERTORA_MASTER_GUIDE.md           ← START HERE
├── CVL_LANGUAGE_DEEP_DIVE.md         ← NEW v1.5: CVL reference
├── VERIFICATION_PLAYBOOKS.md         ← NEW v1.5: Worked examples
├── CERTORA_WORKFLOW.md
├── CERTORA_SPEC_FRAMEWORK.md
├── CERTORA_CE_DIAGNOSIS_FRAMEWORK.md
├── SPEC AUTHORING (CERTORA).md
├── Categorizing_Properties.md
├── BEST_PRACTICES_FROM_CERTORA.md
├── ADVANCED_CLI_REFERENCE.md         ← NEW v1.4: CLI & performance
├── QUICK_REFERENCE_v1.3.md
├── INDEX.md
├── VERSION_HISTORY.md
├── CERTORA_QUICKSTART_TEMPLATE.md
├── TUTORIAL_EXTRACTION_SUMMARY.md
├── POC_TEMPLATE_Foundry.md
├── POC_TEMPLATE_HARDHAT.md
├── VULNERABILITY_REPORT_TEMPLATE.md
│
├── ═══ YOUR CONTRACTS ════════════════════════════════════════════
├── contracts/
│   └── YourContract.sol              ← Primary target
│
├── ═══ SPEC AUTHORING (Analysis) ═════════════════════════════════
├── spec_authoring/
│   ├── yourcontract_spec_authoring.md
│   ├── yourcontract_candidate_properties.md
│   └── yourcontract_causal_validation.md
│
├── ═══ CERTORA (Verification) ════════════════════════════════════
└── certora/
    ├── specs/
    │   ├── validation_yourcontract.spec  ← Run FIRST
    │   └── YourContract.spec             ← Real verification
    ├── confs/
    │   ├── validation_yourcontract.conf
    │   └── YourContract.conf
    └── harnesses/
        └── DummyERC20.sol                ← Simplified externals
```

---

## The Validation-First Approach

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. Write VALIDATION spec first (validates your modeling)              │
│   2. Run: certoraRun validation.conf                                    │
│   3. ALL PASS? → Proceed to real spec                                   │
│   4. Copy infrastructure (methods, ghosts, hooks) from validation       │
│   5. Add REAL invariants and rules from candidate_properties.md         │
│                                                                          │
│   Why? Validation eliminates the HARDEST bugs:                          │
│   ✅ Ghost desync, missing hooks, wrong types, DISPATCHER issues        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chat Prompts (Section 13 in MASTER_GUIDE)

When working with an AI assistant, use these prompts:

| Phase | Prompt Section |
|-------|----------------|
| New project | 13.1 |
| Phase 0/-1 | 13.2 |
| Phase 2 (Properties) | 13.3 |
| Phase 3.5 (Validation) | 13.4 |
| Phase 7 (Real Spec) | 13.5 |
| Debug CEs | 13.6 |

---

## Golden Rule

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   UNDERSTAND → ENUMERATE → VALIDATE → WRITE → DEBUG              │
│                                                                  │
│   ❌ Never write CVL before completing phases 0 through 3.5     │
│   ❌ Never skip causal validation                                │
│   ❌ Never assume external contracts are "standard"              │
│                                                                  │
│   A passing spec means nothing if the modeling is wrong.        │
│   Enumerate reality first. Prove safety second.                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

---

## How to Use the Framework (Quick Start)

### For New Users:
1. **Start with** [INDEX.md](INDEX.md) for navigation
2. **Read** [README.md](README.md) (this file) for overview
3. **Follow** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) step-by-step

### For Returning Users:
1. **Check** [VERSION_HISTORY.md](VERSION_HISTORY.md) for what's new
2. **Use** [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) as cheat sheet
3. **Reference** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) for techniques

### During Verification:
- **Phase 2:** Use `Categorizing_Properties.md` + `BEST_PRACTICES_FROM_CERTORA.md` Section 1
- **Phase 7:** Use `CERTORA_SPEC_FRAMEWORK.md` + `BEST_PRACTICES_FROM_CERTORA.md` Sections 3-5
- **Debugging:** Use `CERTORA_CE_DIAGNOSIS_FRAMEWORK.md` + `BEST_PRACTICES_FROM_CERTORA.md` Section 2

---

## Version

**Version:** 1.5 (RareSkills Integration)  
**Last Updated:** February 2026

### Changelog
- **v1.5:** RareSkills Certora Book integration (35 chapters), CVL Language Deep Dive, Verification Playbooks (ERC-20/WETH/ERC-721), vacuous truth defense, requireInvariant lifecycle, Liveness/Effect/No-Side-Effect pattern, ghost havocing diagnosis
- **v1.4:** Performance optimization, Advanced CLI Reference
- **v1.3:** Property prioritization (HIGH/MEDIUM/LOW), Tutorial best practices integration, 5-step CE investigation, Invariant patterns, Loop handling, Navigation index
- **v1.2:** Dual Mindset approach (should always/never), Test Mining for property discovery
- **v1.1:** Section 9.0 (Validation to Real Spec transition), Section 13 (Chat Prompts)
- **v1.0:** Initial framework release
