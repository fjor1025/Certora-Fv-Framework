# Certora Formal Verification Framework

> **A complete, reusable framework for formal verification of Solidity smart contracts using Certora Prover**  
> **Version:** 2.0 (Validation Evidence Gate)

---

## What's New in v2.0

**Validation Evidence Gate — Structured Proof That Validation Passed Correctly:**

The framework had a critical gap: after `certoraRun` shows ALL PASS, the engineer self-certified "PASSED" and jumped to writing real rules. But "ALL PASS" ≠ "CORRECTLY PASSED." v2.0 adds a mandatory **Validation Evidence Review** between Phase 3.5 and Phase 7.

- **Satisfy Witness Inspection:** Every `satisfy` rule's witness must be inspected for non-degeneracy. A witness with `amount=0` or unchanged state proves nothing meaningful — the function "ran" but did nothing.
- **Ghost Sync Witness Inspection:** Every ghost sync rule must show the ghost at a non-trivial value. `ghost=0 == storage=0` can mean the hook never fired (dead ghost).
- **Mutation Path Completeness:** Mutation path whitelists are cross-referenced against Phase 0 entry points. If a function CAN modify the variable but isn't in the whitelist, the model is incomplete.
- **Advanced Sanity Run:** Required re-run with `rule_sanity: advanced` to catch partial vacuity that `basic` misses.
- **Evidence Sign-Off:** Complete evidence artifact recorded in `causal_validation.md` with Prover job URL, witness inspections, and reviewer signature.

- New Section 7.5 (Validation Evidence Review) in `certora-master-guide.md`
- New Section 13.4.1 (Validation Evidence Review chat prompt)
- Updated Phase 6 Sanity Gate with 6 new evidence checklist items
- Updated FINAL CHECKLIST with evidence requirements

**Previous releases (v1.6–v1.9):**
- **v1.9:** Red Team Hardening (satisfy annotation, failure-path reachability, custom summary accuracy, invariant cycle detection)
- **v1.8:** Reachability Validation (`satisfy` in validation spec, Phase 0.5)
- **v1.7:** Prover v8.8.0 builtin rules (`uncheckedOverflow`, `safeCasting`, `--assume_no_casting_overflow`, `--method` name-only filtering)
- **v1.6:** Revert/failure-path coverage (`@withrevert`, biconditional `<=>`, MUST REVERT WHEN, SILENT PASS diagnosis)

See [version-history.md](version-history.md) for the complete changelog.

---

## Previous Enhancements

**v1.9:** Red Team Hardening (Principal Engineer Audit Fixes)  
**v1.8:** Reachability Validation (satisfy in validation spec, Phase 0.5)  
**v1.7.1:** Quick Start Chat Prompts updated for v1.6/v1.7  
**v1.7:** Prover v8.8.0 Built-in Rules (uncheckedOverflow, safeCasting)  
**v1.6:** Revert/Failure-Path Coverage (@withrevert, biconditional, MUST REVERT WHEN)  
**v1.5:** RareSkills Integration (CVL Deep Dive, Verification Playbooks)  
**v1.4:** Performance Optimization + Advanced CLI Reference  
**v1.3:** Property Prioritization + Tutorial Best Practices + CE Investigation  
**v1.2:** Dual Mindset ("Should Always" / "Should Never") + Test Mining  
**v1.1:** Validation-to-Real-Spec Transition + Chat Prompts

---

## Framework Files

| File | Purpose | When to Use |
|------|---------|-------------|
| **index.md** | Navigation guide & quick access | ← **START HERE for navigation** |
| **certora-master-guide.md** | Complete step-by-step instructions | ← **START HERE for verification** |
| **cvl-language-deep-dive.md** | Complete CVL language reference | ← **NEW in v1.5** - CVL mastery |
| **verification-playbooks.md** | Worked examples (ERC-20, WETH, ERC-721) | ← **NEW in v1.5** - Copy & adapt |
| **advanced-cli-reference.md** | Performance & advanced flags | ← **NEW in v1.4** - Timeouts/optimization |
| **certora-workflow.md** | Phase overview & checklist | Quick reference |
| **certora-spec-framework.md** | CVL 2.0 syntax & templates | Writing actual CVL |
| **certora-ce-diagnosis-framework.md** | Counterexample debugging | When rules fail |
| **spec-authoring-certora.md** | Deep methodology & theory | Understanding WHY |
| **categorizing-properties.md** | Property discovery guidance | Phase 2 |
| **best-practices-from-certora.md** | Official tutorial techniques | Property discovery & patterns |
| **quick-reference-v1.3.md** | Printable cheat sheet | Keep open while coding |
| **version-history.md** | Version tracking & migration | Check what changed |

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
            └── Builtin Safety Scan (uncheckedOverflow, safeCasting) ← v1.7
Phase -1  → Execution Closure (external contracts, modeling decisions)
Phase 2   → Property Discovery (DUAL MINDSET: should always + should never)
            └── Mine tests for invariants, threats, blind spots
            └── Enumerate MUST REVERT WHEN conditions ← v1.6
Phase 2.5 → Classification (INVARIANT vs RULE)
Phase 3.5 → Causal Validation ← RUN VALIDATION SPEC FIRST
            └── satisfy reachability rules (anti-vacuity) ← v1.8
            └── Mutation path + ghost sync validation
Phase 4-6 → Modeling & Sanity Gate
Phase 7   → Write Real CVL Spec
            └── @withrevert + biconditional <=> for every function ← v1.6
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

See `categorizing-properties.md` sections 5 and 6 for details.

---

## Project Structure After Setup

```
your-project/
│
├── ═══ FRAMEWORK FILES ═══════════════════════════════════════════
├── certora-master-guide.md           ← START HERE
├── cvl-language-deep-dive.md         ← NEW v1.5: CVL reference
├── verification-playbooks.md         ← NEW v1.5: Worked examples
├── certora-workflow.md
├── certora-spec-framework.md
├── certora-ce-diagnosis-framework.md
├── SPEC AUTHORING (CERTORA).md
├── categorizing-properties.md
├── best-practices-from-certora.md
├── advanced-cli-reference.md         ← NEW v1.4: CLI & performance
├── quick-reference-v1.3.md
├── index.md
├── version-history.md
├── certora-quickstart-template.md
├── tutorial-extraction-summary.md
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
│   2. Write satisfy reachability rules (proves functions are LIVE) ← v1.8│
│   3. Run: certoraRun validation.conf                                    │
│   4. ALL PASS? → Proceed to real spec                                   │
│   5. Copy infrastructure (methods, ghosts, hooks) from validation       │
│   6. Add REAL invariants and rules from candidate_properties.md         │
│   7. Write @withrevert + biconditional rules for all functions ← v1.6  │
│                                                                          │
│   Why? Validation eliminates the HARDEST bugs:                          │
│   ✅ Always-reverting functions (vacuity) — caught by satisfy ← v1.8   │
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
1. **Start with** [index.md](index.md) for navigation
2. **Read** [readme.md](readme.md) (this file) for overview
3. **Follow** [certora-master-guide.md](certora-master-guide.md) step-by-step

### For Returning Users:
1. **Check** [version-history.md](version-history.md) for what's new
2. **Use** [quick-reference-v1.3.md](quick-reference-v1.3.md) as cheat sheet
3. **Reference** [best-practices-from-certora.md](best-practices-from-certora.md) for techniques

### During Verification:
- **Phase 2:** Use `categorizing-properties.md` + `best-practices-from-certora.md` Section 1
- **Phase 7:** Use `certora-spec-framework.md` + `best-practices-from-certora.md` Sections 3-5
- **Debugging:** Use `certora-ce-diagnosis-framework.md` + `best-practices-from-certora.md` Section 2

---

## Version

**Version:** 1.9 (Red Team Hardening)  
**Last Updated:** February 16, 2026

### Changelog
- **v1.9:** Red Team Hardening — Failure-path reachability (`satisfy lastReverted`), custom summary accuracy protocol (Exact/Over/Under), invariant dependency DAG (`@dev Level: N`), cycle detection, satisfy liveness annotation
- **v1.8:** Reachability Validation — `satisfy` rules as mandatory validation step, Phase 0.5, validation execution order
- **v1.7.1:** Quick Start Chat Prompts updated for v1.6/v1.7 features
- **v1.7:** Prover v8.8.0 builtin rules (`uncheckedOverflow`, `safeCasting`), `--assume_no_casting_overflow`, `--method` name-only
- **v1.6:** Revert/failure-path coverage (`@withrevert`, `<=>` biconditional, MUST REVERT WHEN, SILENT PASS)
- **v1.5:** RareSkills Certora Book integration (35 chapters), CVL Language Deep Dive, Verification Playbooks (ERC-20/WETH/ERC-721), vacuous truth defense, requireInvariant lifecycle, Liveness/Effect/No-Side-Effect pattern, ghost havocing diagnosis
- **v1.4:** Performance optimization, Advanced CLI Reference
- **v1.3:** Property prioritization (HIGH/MEDIUM/LOW), Tutorial best practices integration, 5-step CE investigation, Invariant patterns, Loop handling, Navigation index
- **v1.2:** Dual Mindset approach (should always/never), Test Mining for property discovery
- **v1.1:** Section 9.0 (Validation to Real Spec transition), Section 13 (Chat Prompts)
- **v1.0:** Initial framework release
