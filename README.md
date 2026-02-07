# Certora Formal Verification Framework

> **A complete, reusable framework for formal verification of Solidity smart contracts using Certora Prover**  
> **Version:** 1.4 (Performance-Enhanced + Advanced CLI)

---

## What's New in v1.4

**Performance Optimization & Advanced CLI** (based on Certora Documentation Feb 2026 updates):

**NEW: Advanced CLI Reference Document**:
- `ADVANCED_CLI_REFERENCE.md` - comprehensive guide to performance optimization and advanced flags
- Timeout mitigation strategies (`--split_rules`, `--multi_assert_check`, control flow splitting)
- Advanced debugging (`--multi_example`, `--independent_satisfy`)
- Loop & array handling from Tutorial Lessons 11 & 12
- Multi-version project setup (compiler/optimization/EVM version maps)
- Project-level operations (`--foundry`, `--project_sanity`)
- Harness patterns from Tutorial Lesson 15
- Complete command reference and decision trees

**Updated Documents**:
- `QUICK_REFERENCE_v1.3.md` - Added performance flags section
- `CERTORA_MASTER_GUIDE.md` - Added Section 10.4 (Performance Optimization)
- Framework now includes practical strategies for:
  - Handling timeouts in complex projects
  - Running Foundry fuzz tests with formal verification
  - Managing multi-compiler version projects
  - Optimizing verification performance

**Why This Matters:**
These updates address real-world challenges encountered in production audits and bug bounties. Essential for complex protocols and competitive engagements like Code4rena.

---

## Previous Enhancements

**v1.3:** Property Prioritization + Tutorial Best Practices + CE Investigation  
**v1.2:** Dual Mindset ("Should Always" / "Should Never") + Test Mining  
**v1.1:** Validation-to-Real-Spec Transition + Chat Prompts

---

## Framework Files

| File | Purpose | When to Use |
|------|---------|-------------|
| **INDEX.md** | Navigation guide & quick access | ← **START HERE for navigation** |
| **CERTORA_MASTER_GUIDE.md** | Complete step-by-step instructions | ← **START HERE for verification** |
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
├── CERTORA_WORKFLOW.md
├── CERTORA_SPEC_FRAMEWORK.md
├── CERTORA_CE_DIAGNOSIS_FRAMEWORK.md
├── SPEC AUTHORING (CERTORA).md
├── Categorizing_Properties.md
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

## 🎓 How to Use the Framework (Quick Start)

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

**Version:** 1.3 (Priority-Enhanced + Tutorial Best Practices)  
**Last Updated:** February 5, 2026

### Changelog
- **v1.3:** Property prioritization (HIGH/MEDIUM/LOW), Tutorial best practices integration, 5-step CE investigation, Invariant patterns, Loop handling, Navigation index
- **v1.2:** Dual Mindset approach (should always/never), Test Mining for property discovery
- **v1.1:** Section 9.0 (Validation to Real Spec transition), Section 13 (Chat Prompts)
- **v1.0:** Initial framework release
