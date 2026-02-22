# Framework Version History

> **Current Version:** 3.0 (Offensive Verification + Red Team Hardening)  
> **Last Updated:** February 2026

---

## Version 3.0 (Offensive Verification) - February 2026

### Rationale

The framework excelled at proving correctness of known designs but was **structurally misaligned** for discovering economic exploits. A Principal Engineer red-team review identified that:

1. **No first-class concept of attacker profit** — Zero ghost variables tracking economic position
2. **`satisfy` used for reachability, not attack synthesis** — Never "maximize harm"
3. **Single-transaction mental model** — No multi-step attack patterns
4. **Counterexamples not executable** — No CE → exploit artifact generation
5. **Assertion-centric worldview** — "Prove correct" instead of "prove unexploitable"

v3.0 adds **offensive verification mode** that actively searches for economically impactful exploits before attackers do.

### Philosophy Shift

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FRAMEWORK MISSION                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   OLD (v2.x):                                                            │
│   "Prove the code matches the spec."                                     │
│                                                                          │
│   NEW (v3.0):                                                            │
│   "Assume the design is hostile.                                         │
│    Search for economically profitable counterexamples.                   │
│    Prove safety only AFTER failing to break it."                         │
│                                                                          │
│   START OFFENSIVE. SWITCH TO DEFENSIVE ONLY AFTER EXHAUSTING ATTACKS.   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### New Files

| File | Purpose |
|------|---------|
| **impact-spec-template.md** | Economic impact tracking infrastructure — attacker value ghosts, system value tracking, value extraction counters, anti-invariants, satisfy-based attack search |
| **multi-step-attacks-template.md** | Multi-transaction attack patterns — flash loan, sandwich, staged, governance, reentrancy attack templates |

### New Section: 9.5 Phase 8: Attack Synthesis

Runs in parallel with Phase 7 (after Phase 6 sanity gate passes):

| Subsection | Content |
|-----------|---------|
| **9.5.1 Philosophy Shift** | Defensive vs Offensive verification comparison |
| **9.5.2 Economic Impact Baseline** | Asset enumeration template |
| **9.5.3 Attacker Objective Definition** | CVL patterns for profit tracking |
| **9.5.4 Anti-Invariant Construction** | Rules expected to FAIL (CE = exploit) |
| **9.5.5 Attack Search Process** | Step-by-step guide with `satisfy` |
| **9.5.6 Multi-Step Attack Patterns** | Flash loan, sandwich CVL templates |
| **9.5.7 CE → Exploit Conversion** | Foundry PoC generation workflow |
| **9.5.8 Integration** | How offensive + defensive work together |
| **9.5.9 Phase 8 Checklist** | 11-item completion checklist |

### New CVL Primitives

**Impact Category Ghosts:**
```cvl
persistent ghost mapping(address => mathint) actor_value;  // Per-actor value
persistent ghost mathint total_system_value;                          // Protocol holdings
persistent ghost mathint total_value_extracted;                       // Extraction counter
persistent ghost bool insolvent_state;                                // Insolvency flag
persistent ghost mathint dilution_factor;                             // Share dilution
persistent ghost mapping(address => mathint) socialized_loss;         // Debt spreading
persistent ghost bool liquidity_frozen;                               // Withdrawal blocked
persistent ghost bool irreversible_loss_occurred;                     // Permanent extraction
```

**Anti-Invariant Patterns:**
```cvl
// Rule expected to FAIL — counterexample = exploit parameters
rule attacker_cannot_profit(env e, method f) {
    mathint before = actor_value[e.msg.sender];
    f(e, args);
    mathint after = actor_value[e.msg.sender];
    assert after <= before, "EXPLOIT: Caller profited";
}
```

**Satisfy-Based Attack Search:**
```cvl
rule find_profitable_inputs(env e, method f) {
    mathint profit = actor_value[e.msg.sender]_after - actor_value[e.msg.sender]_before;
    satisfy profit > 0;  // Find exploit parameters
}
```

### Changes by Document

| Document | Changes |
|----------|---------|
| `certora-master-guide.md` | Version 3.0. New §9.5 (Phase 8: Attack Synthesis with 9 subsections). New §13.7 (Phase 8 Chat Prompt). Updated TOC to include Phase 8. Updated §13.8 to include Phase 8 in current phase list. |
| `certora-spec-framework.md` | Version 3.0. New Impact Category Ghosts section. New Impact Tracking Hooks section. New Impact Definitions section. New Anti-Invariant Templates section. New Impact-Driven Invariants section. |
| `certora-workflow.md` | Version 3.0. Updated workflow diagram with Phase 8 nodes. Added Phase 8 to phase summary table. Added impact/attack templates to document reference table. |
| `certora-ce-diagnosis-framework.md` | Version 3.0. New CE→Exploit Conversion section. Updated classification table (REAL BUG → Convert to PoC). New CE→Foundry conversion template. |
| `spec-authoring-certora.md` | Added Phase 8 (Attack Synthesis) section after Phase 7. Offensive verification philosophy and workflow. |
| `categorizing-properties.md` | Version 3.0. New §0 Economic Impact Categories. Impact taxonomy table. Impact property template. Attacker objective checklist. |
| `readme.md` | Version 3.0. New Framework Mission section. New What's New in v3.0 section. Updated Framework Files table with new documents. |
| `index.md` | Version 3.0. New Offensive Verification (Phase 8) section. Updated Core Learning Path with impact template. |
| `version-history.md` | This entry. |

### Migration from v2.0

1. **Add impact-spec-template.md** to your verification workflow
2. **Add multi-step-attacks-template.md** for DeFi protocols
3. **After Phase 6 sanity gate passes**, run Phase 8 offensive verification IN PARALLEL with Phase 7
4. **Run anti-invariants** expecting them to fail
5. **Run hook liveness checks** before trusting anti-invariant results
6. **Convert any CEs** to Foundry PoCs for validation

### Key Concept: Offensive-First Verification (v3.0)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OFFENSIVE-FIRST VERIFICATION                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Phases 0-6: Analysis & Modeling                                        │
│      Map execution surface → Discover properties → Validate closure      │
│                                                                          │
│   After Phase 6 sanity gate passes, run IN PARALLEL:                     │
│                                                                          │
│   DEFENSIVE (Phase 7):              OFFENSIVE (Phase 8):                 │
│   Write correctness invariants      Write anti-invariants (expect FAIL)  │
│   Prove properties hold             Run profit search + liveness checks  │
│   Debug violations                  Run multi-step attack patterns       │
│                                     If FAIL → EXPLOIT FOUND              │
│                                     If PASS → No attack (or incomplete)  │
│                                                                          │
│   COMPLETE: Both defensive AND offensive verification pass               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Version 2.0 (Validation Evidence Gate) - February 2026

### Rationale
The framework's Phase 3.5 → Phase 7 transition had a critical gap: the engineer self-certified "ALL PASS" without structured verification that the validation actually passed *correctly*. A satisfy rule can "pass" by finding a degenerate witness (amount=0, balance unchanged). A ghost sync rule can "pass" trivially (0 == 0 because the hook never fires). A mutation path rule can pass while missing functions. The `rule_sanity: basic` setting doesn't catch partial vacuity. Without evidence review, the real spec is built on unverified infrastructure.

### The Problem This Solves

```
┌─────────────────────────────────────────────────────────────────────────┐
│ WITHOUT Validation Evidence Review:                                      │
│                                                                          │
│   certoraRun validation.conf → ALL PASS (green dashboard)               │
│        ↓                                                                 │
│   Engineer says "PASSED!" and proceeds to Phase 7                       │
│        ↓                                                                 │
│   But satisfy found degenerate witness (amount=0, no state change)      │
│   Ghost sync passed trivially (0 == 0, hook never fired)                │
│   Mutation path rule missed function3 (incomplete whitelist)             │
│   rule_sanity: basic didn't catch partial vacuity                       │
│        ↓                                                                 │
│   Real spec is built on HOLLOW infrastructure.                          │
│   Every rule may be vacuously true.                                     │
│                                                                          │
│ WITH Validation Evidence Review (v2.0):                                  │
│                                                                          │
│   certoraRun validation.conf → ALL PASS                                 │
│        ↓                                                                 │
│   Engineer INSPECTS satisfy witnesses → finds amount=0 → STOPS          │
│   Engineer INSPECTS ghost sync → finds ghost=0 before/after → STOPS     │
│   Engineer CROSS-REFERENCES mutation whitelist vs Phase 0 → finds gap   │
│   Engineer RUNS rule_sanity: advanced → catches hidden vacuity          │
│        ↓                                                                 │
│   Fixes infrastructure BEFORE writing real rules.                       │
│   Signs off evidence artifact. Proceeds with confidence.                │
└─────────────────────────────────────────────────────────────────────────┘
```

### New Section: 7.5 Validation Evidence Review

Added between Phase 3.5 "Run Validation" (7.4) and Phase 4-6 (Modeling & Sanity):

| Subsection | Content |
|-----------|---------|
| **7.5.1 Evidence Collection Template** | Complete markdown template for `causal_validation.md` — Rule Status Table, Satisfy Witness Inspection, Ghost Sync Witness Inspection, Mutation Path Completeness Check, Advanced Sanity Run, Sign-Off |
| **7.5.2 Common Degenerate Witnesses** | 6-row diagnostic table: amount=0, from==to, ghost=0, single function, same address, msg.value for non-payable |
| **7.5.3 STOP vs PROCEED Decision Matrix** | Clear criteria for when to halt (degenerate/trivial/incomplete/failed) vs when Phase 7 is safe |

### New Section: 13.4.1 Validation Evidence Review Chat Prompt

Added between 13.4 (Phase 3.5 Causal Validation) and 13.5 (Phase 7 Write Real Spec):
- 5-step structured review process (satisfy witnesses, ghost sync, mutation completeness, advanced sanity, sign-off)
- Gate criteria: all 5 must pass before Phase 7 entry is permitted
- References to Section 7.5 templates and diagnostic tables

### Changes by Document

| Document | Changes |
|----------|---------|
| `certora-master-guide.md` | Version 2.0. New §7.5 (Validation Evidence Review with 3 subsections). Updated §7.4 (STOP warning after ALL PASS). Updated validation spec WORKFLOW comment (5 steps, evidence before real spec). Updated §7.1 checklist (5 new evidence items). Updated §8.3 Sanity Gate (6 new evidence checklist items). New §13.4.1 chat prompt. Updated §13.5 (added Evidence Review prerequisite field). Updated §9.0 (added prerequisite warning). Updated FINAL CHECKLIST (2 new items). |
| `readme.md` | Version 2.0. Updated What's New section. |
| `version-history.md` | This entry. |
| `certora-quickstart-template.md` | Version bump to 2.0. |
| `certora-workflow.md` | Version bump to 2.4. |
| `certora-ce-diagnosis-framework.md` | Version bump to 2.3. |
| `certora-spec-framework.md` | Version bump to 2.2. |
| `cvl-language-deep-dive.md` | Version bump to v2.0. |
| `verification-playbooks.md` | Version bump to v2.0. |
| `advanced-cli-reference.md` | Version bump to Framework v2.0. |
| `index.md` | Version bump to v2.0. |

### Key Concept: Evidence-Based Validation (v2.0)

```
Previous Flow (v1.9):
  certoraRun → ALL PASS → Phase 7 (self-certified)

New Flow (v2.0):
  certoraRun → ALL PASS → Evidence Review → Sign-Off → Phase 7

  Evidence Review:
  1. Inspect satisfy witnesses  →  Non-degenerate? (amount > 0, state changes)
  2. Inspect ghost sync witnesses → Non-trivial? (ghost ≠ 0, hook fires)
  3. Cross-ref mutation whitelists → Complete? (matches Phase 0 entry points)
  4. Re-run rule_sanity: advanced → No hidden vacuity?
  5. Sign off in causal_validation.md → Accountability trail
```

---

## Version 1.9 (Red Team Hardening — Principal Engineer Audit Fixes) - February 2026

### Rationale
The framework was subjected to a full "Principal Engineer's Audit" — a Red Team review that assumed the framework was flawed until proven solid, testing it against 4 Pillars of Failure: Vacuity (Silent Pass), Ghost Integrity (Harness), Revert Precision, and Invariant Lifecycle. Four findings were identified and fixed.

### Finding P1-A: Satisfy Proves Liveness, Not Effect (LOW)
**Problem:** The `satisfy !lastReverted` template could be misinterpreted as proving that a function *works correctly*, when it only proves a non-reverting path *exists*. A function that runs but does nothing (e.g., `withdraw(0)`) passes this check.

**Fix:** Added explicit annotation to the Validation Rule 0 template clarifying that `satisfy` proves liveness, mutation path rules prove effect, and both are required.

**Files changed:** `certora-master-guide.md` (§7.2 Validation Rule 0 comment block)

### Finding Q2: No Failure-Path Satisfy Template (LOW)
**Problem:** The framework validated success-path reachability (`satisfy !lastReverted`) but had no explicit template for validating that revert paths are reachable (`satisfy lastReverted`). Without this, a biconditional `<=>` revert rule can pass vacuously because the failure scenario is impossible under the modeling.

**Fix:** Added **Validation Rule 0b: Failure-Path Reachability** with complete templates and checklist items.

**Files changed:**
- `certora-master-guide.md` (§7.1 checklist, §7.2 new Rule 0b template, §8.3 Sanity Gate)
- `spec-authoring-certora.md` (Phase 6 Causal Closure Checks)
- `best-practices-from-certora.md` (§7.3 item #7 with worked example)

### Finding P2-B: Custom Summary Accuracy Validation (MEDIUM)
**Problem:** NONDET and DISPATCHER summaries had explicit checklists, but custom summaries (ghost-based uninterpreted functions) were trusted implicitly with no accuracy validation. A custom summary that overpromises determinism can hide real bugs.

**Fix:** Added mandatory accuracy annotation (Exact / Overapproximation / Underapproximation) and justification requirement for all custom summaries. Added dedicated checklist section.

**Files changed:**
- `certora-spec-framework.md` (Custom Summary Rules section — added validation protocol and annotated example)
- `spec-authoring-certora.md` (Phase 6 — new Custom Summary Accuracy Checks subsection)
- `certora-master-guide.md` (§8.3 Sanity Gate — new Custom Summary Accuracy checklist)

### Finding P4-A: Invariant Cycle Detection Protocol (MEDIUM)
**Problem:** `requireInvariant` cycle detection was manual. In a large spec with 20+ invariants, an engineer could create A→B→A circular dependencies without noticing. The Prover does NOT reject cyclic `requireInvariant` chains — it simply assumes both hold, creating a logical loop.

**Fix:** Introduced a formal Cycle Detection Protocol:
1. Every invariant must have `@dev Level: N` annotation
2. `requireInvariant` may only reference strictly lower-level invariants
3. Base invariants (Level 1) must be proven in isolation first (`--rule`)
4. Dependency DAG must be documented in `causal_validation.md`
5. If you cannot assign a level, you have a cycle → refactor

**Files changed:**
- `certora-spec-framework.md` (Invariants template section — added DAG protocol, updated all `@dev` tags)
- `certora-ce-diagnosis-framework.md` (Phase -1.4 Invariant Dependencies — expanded checklist + protocol)
- `best-practices-from-certora.md` (§8.4 new Circular Dependency Detection section)
- `certora-master-guide.md` (§8.3 Sanity Gate — new Invariant Dependency Safety checklist)
- `spec-authoring-certora.md` (Phase 6 — new Invariant Dependency Safety subsection)

### Summary of Changes by Document

| Document | Changes |
|----------|---------|
| `certora-master-guide.md` | Rule 0 annotation, Rule 0b template, checklist additions (3 new sections in Sanity Gate) |
| `certora-spec-framework.md` | Custom summary validation protocol, invariant DAG protocol, `@dev Level: N` tags |
| `certora-ce-diagnosis-framework.md` | Expanded Phase -1.4 with cycle detection checks |
| `spec-authoring-certora.md` | Phase 6 — 3 new checklist subsections |
| `best-practices-from-certora.md` | Satisfy annotation, failure-path §7.3#7, cycle detection §8.4 |
| `readme.md` | Version bump to 1.9, changelog entry |
| `version-history.md` | This entry |

---

## Version 1.8 (Reachability Validation — `satisfy` in Validation Spec) - February 2026

### Rationale
The framework had `satisfy` documented as a *reference concept* (cvl-language-deep-dive §4) and a *reactive diagnostic tool* (ce-diagnosis-framework B.5), but it was **never prescribed as a mandatory proactive step in the validation workflow**. This is the single most important gap: without `satisfy` reachability rules, a validation spec can pass all assertion rules while proving nothing — because every function always reverts (vacuity). This is the "Gold Standard" gap identified by senior specification architects: **before proving any assert, prove each function CAN execute without reverting**.

### The Problem This Solves

```
┌─────────────────────────────────────────────────────────────────────────┐
│ WITHOUT reachability validation:                                         │
│                                                                          │
│   withdraw() always reverts (bad harness / require / DISPATCHER)        │
│        ↓                                                                 │
│   rule deposit_withdraw_integrity { ... withdraw(e, amount); ... }      │
│        ↓                                                                 │
│   assert balance_after == balance_before - amount  →  ✅ PASSES         │
│        ↓                                                                 │
│   But "after" state NEVER EXISTS → assert is VACUOUSLY TRUE             │
│   You verified a lie.                                                    │
│                                                                          │
│ WITH reachability validation (v1.8):                                     │
│                                                                          │
│   rule validation_reachability_withdraw() {                              │
│       withdraw@withrevert(e, amount);                                    │
│       satisfy !lastReverted;  →  ❌ VIOLATED (function always reverts)  │
│   }                                                                      │
│        ↓                                                                 │
│   STOP. Fix harness before writing any assert rules.                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Changes by Document

#### certora-master-guide.md
- **§7.2 Validation Spec Template**: Added **Validation Rule 0: Function Reachability** with complete `satisfy` pattern — placed BEFORE mutation path and ghost sync rules
- **§7.1 Causal Validation Checklist**: Added 2 new items (satisfy rules written + all pass)
- **§8.3 Phase 6 Sanity Gate**: Added reachability gate to Causal Closure section
- **§9.0 What Validation Passing Guarantees**: Added "Always-reverting functions (vacuity)" to eliminated bugs list
- **§9.0 What to KEEP, DELETE, ADD**: Added `rule validation_reachability_*` → DELETE row
- **§13.4 Phase 3.5 Prompt**: Added satisfy step (#3), validation execution order guidance, Phase 0.5 concept
- **FINAL CHECKLIST**: Added "Function reachability (satisfy) PASSED"

#### spec-authoring-certora.md
- **Phase 6 Causal Closure Checks**: Added satisfy reachability gate

#### certora-spec-framework.md
- **Causal Closure Checklist**: Added satisfy reachability row as first item

#### verification-playbooks.md
- **§4.1 Four-Phase System**: Added **Phase 0.5: REACHABILITY VALIDATION** between builtin scan and function correctness

#### best-practices-from-certora.md
- **§7.3 Defenses**: Added item #6 with complete worked `satisfy` rule example and explanation

### Key Concept: Validation Execution Order (v1.8)

```
1. satisfy rules (reachability)  →  Proves functions are LIVE
2. Mutation path rules (assert)  →  Proves modeling is COMPLETE
3. Ghost sync rules (assert)     →  Proves ghosts track REALITY
4. ALL PASS                      →  Safe to write real spec
```

---

## Version 1.7.1 (Quick Start Prompts Update) - February 2026

### Rationale
Section 13 (Quick Start Chat Prompts) in `certora-master-guide.md` was still written for v1.5 and did not reflect any of the v1.6 (revert/failure-path coverage) or v1.7 (Prover v8.8.0 builtin rules) additions. Every prompt template was updated to surface these capabilities so AI-assisted verification sessions use the full framework from the start.

### Changes (certora-master-guide.md Section 13)

| Prompt | v1.6 Additions | v1.7 Additions |
|--------|----------------|----------------|
| **13.1** New project | Revert coverage refs, Pattern B reference | Prover version field, builtin scan step, `unchecked{}` field |
| **13.2** Phase 0/-1 | Revert condition documentation per call | Builtin safety scan, §19.1 reference |
| **13.3** Phase 2 | MUST REVERT WHEN property category, REVERT field | — |
| **13.3.1** Dual mindset | MUST REVERT WHEN enumeration, revert test mining | — |
| **13.4** Phase 3.5 | @withrevert revert validation rules | `use builtin rule sanity;` step |
| **13.5** Phase 7 | @withrevert step (#7), Pattern B reference | `use builtin rule` step (#8), §19.1 reference |
| **13.5.1** Token standard | @withrevert for all functions, safeMint/safeTransferFrom | Phase 0 builtin scan, overflow/casting refs |
| **13.6** Debugging CEs | SILENT PASS diagnosis (5 new questions) | — |
| **13.7** Essential info | — | Prover version + unchecked{}/assembly fields |

### Version labels consolidated
- Removed stale "NEW v1.3" / "NEW v1.5" labels from prompt bodies
- Replaced with current "← NEW v1.6" / "← NEW v1.7" markers

---

## Version 1.7 (Prover v8.8.0 Built-in Rules) - February 2026

### Rationale
Certora Prover v8.8.0 introduced powerful built-in rules (`uncheckedOverflow`, `safeCasting`) that automate detection of two critical vulnerability classes — unsafe unchecked arithmetic and silent casting truncation — without writing any custom rules. The framework needed to integrate these as first-class verification steps.

### New Prover Features Covered

| Feature | Type | What It Does |
|---------|------|-------------|
| `use builtin rule uncheckedOverflow` | Builtin rule | Auto-checks all `+`, `-`, `*` in `unchecked` blocks for overflow/underflow |
| `use builtin rule safeCasting` | Builtin rule | Auto-checks all explicit casts for out-of-bounds truncation |
| `--assume_no_casting_overflow` | CLI flag | Assumes all casts are in bounds (underapproximation — use after `safeCasting` verification) |
| `--method` / `--exclude_method` name-only | CLI change | Name without parameter types now matches all overloads |

### Changes by Document

#### cvl-language-deep-dive.md
- Added **Section 19.1: Built-in Rules — Automated Safety Checks** with:
  - Complete 7-rule reference table (sanity, deepSanity, msgValueInLoopRule, hasDelegateCalls, viewReentrancy, safeCasting, uncheckedOverflow)
  - Detailed `uncheckedOverflow` documentation with example and .conf requirements
  - Detailed `safeCasting` documentation with cast categories (unsigned, signed/unsigned, signed)
  - `--assume_no_casting_overflow` documentation with safety warnings
  - Recommended builtin rule strategy (Phase 0 vs Phase 1)

#### advanced-cli-reference.md
- Updated `--method` examples with name-only filtering syntax (v8.8.0+)
- Updated `--exclude_method` with name-only support
- Updated CLI decision tree with name-only hint

#### certora-spec-framework.md
- **Revert/Failure-Path Checklist**: Added 2 items for `uncheckedOverflow` and `safeCasting` builtin rules

#### verification-playbooks.md
- **Four-Phase System**: Added "Phase 0: BUILTIN SAFETY SCAN" with all 6 builtin rules
- **Common Pitfalls**: Added unchecked overflow and silent cast truncation entries

#### best-practices-from-certora.md
- **Pre-Run Prover Checklist**: Added 2 items for `uncheckedOverflow` and `safeCasting`

---

## Version 1.6 (Revert/Failure-Path Coverage) - June 2025

### Rationale
The framework had strong revert/failure-path coverage in reference documents (cvl-language-deep-dive.md §5, verification-playbooks.md §4.3) but weak coverage in process/workflow documents. A specification engineer following the phases sequentially would not be guided to verify revert behavior until discovering it in reference material. This release promotes revert verification from "technique you might find" to "mandatory process step."

### Principle
> **Proving why something reverts is just as important as proving success.**  
> By default, the Certora Prover ignores all execution paths that revert. If you don't use `@withrevert`, revert paths are silently pruned — a passing rule only proves the happy path.

### Changes by Document

#### certora-spec-framework.md
- **Phase 3 Rule Implementation Pattern**: Expanded from 1 generic pattern to 3 explicit patterns:
  - Pattern A: Success-Path-Only (with mandatory companion note)
  - Pattern B: Complete with `@withrevert` + Liveness/Effect/No-Side-Effect (marked PREFERRED)
  - Pattern C: Dedicated Revert Rule
- Added critical warning block about default revert pruning behavior
- **Phase 4**: Added "Revert/Failure-Path Checklist" (7 items) before Quality Checklist

#### spec-authoring-certora.md
- **Phase 3 Property Specification Block**: Added "Revert/Failure Behavior" field requiring documentation of success/revert conditions for each function
- **Phase 6 Final Sanity Gate**: Added "Revert/Failure Coverage Checks" section (7 checklist items)

#### certora-master-guide.md
- **Final Checklist**: Added 3 new items: Revert Coverage, Liveness Assertions, No Silent Revert Pruning
- **Quick Reference §12.2**: Added revert verification syntax examples (`@withrevert`, `bool success`, biconditional patterns)

#### best-practices-from-certora.md
- **Pre-Run Prover Checklist**: Added 4 new items for revert coverage (`@withrevert` usage, `<=>` assertions, `lastReverted` capture timing, failure condition documentation)

#### categorizing-properties.md
- Added "Revert Behavior Checklist (MUST REVERT WHEN)" section with 6 diagnostic questions
- Enhanced property documentation template with `REVERT (Must Revert When)` field
- Enhanced example property (A3. Withdrawal Integrity) with concrete revert conditions
- Added 2 new entries to Common Mistakes table (revert conditions, SHOULD ALWAYS-only properties)

#### certora-ce-diagnosis-framework.md
- Added **SILENT PASS** classification to the CE classification table
- Added "Silent Pass" warning block explaining how pruned revert paths cause false confidence
- Added concrete before/after code example demonstrating the problem

### Documents Unchanged (Already Adequate)
- **cvl-language-deep-dive.md**: Section 5 already provides comprehensive `@withrevert`/`lastReverted` reference with biconditional patterns, overwriting warnings, and non-payable/overflow traps
  - *v1.6 addition:* Built-in max constants table (`max_uint8` through `max_uint256`) added to Section 1
- **verification-playbooks.md**: Already uses `@withrevert` in every function rule, uses Liveness/Effect/No-Side-Effect pattern consistently, has Revert Condition Enumeration Checklist in §4.3
  - *v1.6 additions:* SafeMint (§3.11) and SafeTransferFrom (§3.12) callback-aware rules, Initializable Playbook (§5), Nonces Playbook (§6)

---

## Version 1.5 (RareSkills Integration) - February 2026

### Knowledge Source
Complete analysis of the **RareSkills Certora Book** — 35 chapters, 60,000+ words, produced in collaboration with Certora. Every chapter was systematically read and knowledge extracted to fill 19 identified gaps in the framework.

### Major Additions

#### New Documents

1. **cvl-language-deep-dive.md** — Complete CVL language reference (20 sections)
   - **Section 1: Type System** — `mathint` semantics, `require_uint256` dangers, `to_mathint`/`assert_uint256` casting reference
   - **Section 2: Core Statements** — `require`, `assert`, `satisfy` with existential quantification and solver examples
   - **Section 3: Logical Operators** — `=>` implication, `<=>` biconditional, contrapositive equivalence
   - **Section 4: Vacuous Truth & Tautology** — How false preconditions silently pass all assertions; defense strategies
   - **Section 5: Method Tags** — `@withrevert`, `@norevert`, `lastReverted` capture patterns, biconditional revert enumeration
   - **Section 6: Environment Variables** — `env`, `msg.sender`, `msg.value`, non-payable trap, overflow trap
   - **Section 7: Native Balances** — `nativeBalances[address]`, `currentContract`, self-call exclusion
   - **Section 8: Ghost Variables** — Declaration, havocing behavior, `init_state axiom` vs `axiom`, persistent ghosts
   - **Section 9: Hooks** — Sstore delta pattern, Sload constraint pattern, CALL opcode hook for assembly
   - **Section 10: `definition` Blocks** — Reusable CVL expressions, standard definitions library
   - **Section 11: Invariants** — Base case + inductive step proof structure, preserved blocks (generic, function-specific, constructor)
   - **Section 12: `requireInvariant` Lifecycle** — `require` → prove invariant → `requireInvariant` upgrade path
   - **Section 13: Parametric Rules** — `method f`, `calldataarg`, method properties (`f.isView`, `f.selector`), filtered blocks
   - **Section 14: Partially Parametric Rules** — `helperSoundFnCall` routing pattern for per-function preconditions
   - **Section 15: Liveness/Effect/No-Side-Effect** — OpenZeppelin's industry-standard three-part assertion pattern
   - **Section 16: DISPATCHER & Callbacks** — Mock receiver pattern, `DISPATCHER(true)` for external callbacks
   - **Section 17: Loops** — Hidden string loops, `--loop_iter`, `--optimistic_loop` tradeoffs
   - **Section 18: Self-Transfer** — `from == to` edge case, compact `assert_uint256` pattern, harness functions
   - **Section 19: Invariant Sanity Checks** — `rule_not_vacuous`, `invariant_not_trivial_postcondition`
   - **Section 20: Quick Reference Tables** — Statement semantics, operator reference, ghost initialization, hook types

2. **verification-playbooks.md** — Complete worked verification examples
   - **ERC-20 Playbook (22 rules):**
     - Phase 1: Function correctness (transfer, transferFrom, approve, mint, burn)
     - Phase 2: No side effects (balance isolation, allowance isolation)
     - Phase 3: Global invariants (sum tracking, individual cap, zero-address)
     - Phase 4: Authorization (supply governance, balance governance, allowance governance)
     - Ghost + hook infrastructure (sum tracking with Sstore + Sload)
     - Complete certoraRun configuration
   - **WETH Playbook (Solady Pattern):**
     - Solvency invariant (`nativeBalances[currentContract] >= totalSupply()`)
     - Deposit/withdraw verification with persistent ghost for CALL opcode
     - Non-standard storage layout handling (Solady computed slots)
     - Self-call exclusion modeling
   - **ERC-721 Playbook (OpenZeppelin Pattern):**
     - Harness contract pattern (`unsafeOwnerOf`, `unsafeGetApproved`)
     - Mock receiver for DISPATCHER callbacks
     - Ownership tracking via Sstore hook with ternary mint/burn/transfer handling
     - `helperSoundFnCall` routing for per-function preconditions
     - Complete mint/burn/transferFrom with Liveness/Effect/No-Side-Effect
     - Supply change authorization
   - **Methodology Reference:**
     - Four-phase system diagram
     - Ghost + hook setup checklist
     - Revert condition enumeration checklist
     - Persistent ghost decision table
     - Common pitfalls and solutions

#### Updated Documents

3. **certora-spec-framework.md**
   - Added `definition` blocks section (nonpayable, nonzerosender, balanceLimited)
   - Added Liveness/Effect/No-Side-Effect rule template in CVL 2.0 Templates section

4. **best-practices-from-certora.md**
   - Added Section 7: Vacuous Truth & Tautology Defense
   - Added Section 8: The `require` → `requireInvariant` Lifecycle
   - Added Section 9: Self-Transfer & Edge Case Patterns
   - Updated framework phase mapping table
   - Expanded pre-run checklist with v1.5 items

5. **certora-ce-diagnosis-framework.md**
   - Added Ghost Havocing Diagnosis Guide with symptoms, steps, and fixes
   - Added Persistent Ghost + CALL Hook template
   - Added havocing behavior comparison table (regular vs persistent)

6. **certora-master-guide.md**
   - Version bump to v1.5
   - Added cvl-language-deep-dive.md and verification-playbooks.md to document table

7. **readme.md**
   - Version bump to v1.5
   - Complete "What's New in v1.5" section
   - Updated framework files table with new documents
   - Updated changelog

### 19 Gaps Filled

| # | Gap | Resolution |
|---|-----|------------|
| 1 | `satisfy` semantics | CVL Deep Dive §2 |
| 2 | `=>` and `<=>` operators | CVL Deep Dive §3 |
| 3 | Vacuous truth / tautology | CVL Deep Dive §4, Best Practices §7 |
| 4 | `mathint` type system | CVL Deep Dive §1 |
| 5 | `@withrevert` / `lastReverted` | CVL Deep Dive §5 |
| 6 | Liveness/Effect/No-Side-Effect | CVL Deep Dive §15, Playbooks §1.5-§3.8 |
| 7 | Persistent ghosts + CALL hooks | CVL Deep Dive §8-9, CE Diagnosis |
| 8 | `definition` blocks | CVL Deep Dive §10, Spec Framework |
| 9 | Partially parametric rules | CVL Deep Dive §14 |
| 10 | Complete ERC-20 verification | Playbooks §1 |
| 11 | Non-payable handling | CVL Deep Dive §6, §10 |
| 12 | `nativeBalances` | CVL Deep Dive §7 |
| 13 | Self-transfer edge case | CVL Deep Dive §18, Best Practices §9 |
| 14 | Ghost havocing | CVL Deep Dive §8, CE Diagnosis |
| 15 | Global axioms | CVL Deep Dive §8 |
| 16 | `require` → `requireInvariant` | CVL Deep Dive §12, Best Practices §8 |
| 17 | DISPATCHER for callbacks | CVL Deep Dive §16, Playbooks §3.3-3.5 |
| 18 | Hidden loops (strings) | CVL Deep Dive §17 |
| 19 | Invariant sanity checks | CVL Deep Dive §19 |

---

## Version 1.4 (Performance-Enhanced + Advanced CLI) - February 7, 2026

### Major Additions

#### New Documents
1. **advanced-cli-reference.md** - Comprehensive performance optimization and advanced CLI guide
   - **Section 1: Performance Optimization**
     - `--split_rules` strategy for heavy rules
     - `--multi_assert_check` for timeout mitigation
     - Control flow splitting techniques (depth, mediumTimeout, eager/lazy splitting)
     - Complete timeout mitigation checklist
   
   - **Section 2: Advanced Debugging**  
     - `--multi_example` for multiple counterexamples
     - `--independent_satisfy` for separate satisfy checks
     - Expression breakdown techniques
   
   - **Section 3: Loop & Array Handling** (from Tutorial Lessons 11 & 12)
     - `--loop_iter` and `--optimistic_loop` strategies
     - Array uniqueness patterns
     - Frequency tracking with ghosts
     - Array length bounds
   
   - **Section 4: Multi-Version Projects**
     - `--compiler_map` for mixed Solidity/Vyper versions
     - `--solc_optimize_map` for per-contract optimization
     - `--solc_evm_version_map` for different EVM targets
     - `--solc_via_ir_map` for IR pipeline configurations
   
   - **Section 5: Project-Level Operations**
     - `--project_sanity` for quick project assessment
     - `--foundry` for formal verification of Foundry fuzz tests
     - `--ignore_solidity_warnings` for legacy contracts
   
   - **Section 6: Harness Patterns** (from Tutorial Lesson 15)
     - Complete harness template with best practices
     - Harness vs. CVL ghosts decision tree
     - DO/DON'T guidelines
   
   - **Section 7: Practical Tips**
     - `--coverage_info` for gap analysis
     - `--rule_sanity` for vacuity checks
     - Running rules individually strategies
     - Config file best practices
   
   - **Section 8: Quick Command Reference**
     - Performance optimization commands
     - Advanced debugging commands
     - Complete real-world examples

#### Enhanced Documents

**quick-reference-v1.3.md**
- Added **"⚡ Performance & Advanced Flags"** section
  - Timeout mitigation quick reference table
  - Advanced debugging flags table
  - Quick performance commands
- Updated document links to include ADVANCED_CLI_REFERENCE
- Added loop/array and multi-version project references

**certora-master-guide.md**
- Updated to v1.4
- Added ADVANCED_CLI_REFERENCE to framework documents table (Section 1.1)
- Added **Section 10.4: Performance Optimization & Timeout Mitigation**
  - Quick timeout fixes table
  - Common performance commands
  - Performance decision tree
  - Advanced debugging flags
  - Direct reference to detailed strategies in ADVANCED_CLI_REFERENCE

**readme.md**
- Updated to v1.4
- Added comprehensive "What's New in v1.4" section
- Added ADVANCED_CLI_REFERENCE to framework files table
- Emphasized real-world applicability for production audits and bug bounties

### Integration Points

**Tutorial Content Integration:**
- Lesson 11 (Loops): Loop unrolling strategies, `--loop_iter`, `--optimistic_loop` safety
- Lesson 12 (Arrays): Array uniqueness patterns, frequency tracking, length bounds
- Lesson 15 (Harness): Harness template, best practices, harness vs. ghosts decision
- 3-Day Workshop: Practical workflow insights

**Certora Documentation (Feb 2026 Updates):**
- Latest CLI flags and options
- Control flow splitting configurations
- Multi-version project setup patterns
- Foundry integration workflow

### Philosophy

**Framework Evolution:**
Version 1.4 addresses the transition from "learning formal verification" to "production formal verification at scale":

- **v1.1-v1.3** focused on methodology and property discovery
- **v1.4** focuses on performance, productivity, and real-world complexity

This version is designed for:
- Complex DeFi protocols with timeouts
- Multi-contract systems with various compiler versions
- Competitive audit environments (Code4rena, Immunefi)
- Bug bounty hunters seeking formal verification edge
- Production audits requiring comprehensive coverage

---

## Version 1.3 (Tutorial-Enhanced) - February 5, 2026

### Major Additions

#### New Documents
1. **best-practices-from-certora.md** - Comprehensive extraction from official Certora tutorials
   - Property discovery techniques (Lesson 06)
   - 5-step CE investigation workflow (Lesson 02)
   - Invariant design patterns (Lessons 07, 08, 15)
   - Harness best practices (Lesson 15)
   - Loop handling strategies (Lesson 11)
   - Common pitfalls and anti-patterns

2. **quick-reference-v1.3.md** - Printable cheat sheet
   - Phase checklists with best practices
   - Invariant pattern quick reference
   - Pre-verification checklist
   - Command cheat sheet

3. **tutorial-extraction-summary.md** - Documentation of extraction process
   - Tutorial-to-framework mapping
   - Key insights extracted
   - Integration points
   - Usage guidelines

#### Enhanced Documents

**categorizing-properties.md**
- Added **Section 7: Property Prioritization**
  - HIGH / MEDIUM / LOW priority levels
  - Impact-based prioritization matrix
  - Priority assignment methodology
  - Verification time budget allocation
  - Complete property entry template
- Updated checklist to v1.3

**certora-ce-diagnosis-framework.md**
- Updated to v2.1
- Added **Tutorial-Based Investigation Workflow** section
  - 5-step systematic investigation process
  - Call trace analysis checklist (storage, arguments, returns)
  - Expression breakdown best practices
  - Bug documentation template

**certora-master-guide.md**
- Updated to v1.3
- Added BEST_PRACTICES document to framework table
- Enhanced Phase 2 (Property Discovery) with:
  - Reference to best practices Section 1
  - Property prioritization guidance
  - Priority field in property template
  - Priority column in summary tables
- Enhanced Phase 10.3 (CE Debugging) with tutorial workflow reference
- Added reference to common pitfalls (Section 12.4)
- Updated Phase 2 chat prompt with v1.3 enhancements

**certora-workflow.md**
- Updated to v2.1
- Enhanced framework overview diagram with BEST_PRACTICES and updated Categorizing_Properties
- Added v1.3 enhancements note to Phase 2
- Added reference to 5-step CE debugging

**certora-quickstart-template.md**
- Added "What's New in v1.3" section
- References to prioritization, dual mindset, test mining

**readme.md**
- Updated to v1.3
- Added comprehensive "What's New" section
- Added BEST_PRACTICES to file list
- Previous enhancements section updated

---

## Version 1.2 (Dual Mindset Enhanced) - January 31, 2026

### Enhancements

**categorizing-properties.md**
- Added **Section 5: RIGHT / WRONG BEHAVIOR DUAL CHECKLIST**
  - Dual mindset approach: "Should Always" vs "Should Never"
  - Attack vector enumeration
  - Comprehensive threat modeling
  
- Added **Section 6: Mining Properties from Test Suites**
  - Test-to-property translation patterns
  - Coverage blind spot identification
  - Implicit invariant extraction

**certora-master-guide.md**
- Updated Phase 2 chat prompt with dual mindset option (Section 13.3.1)

**readme.md**
- Added v1.2 features section

---

## Version 1.1 (Transition Enhanced) - January 27, 2026

### Enhancements

**certora-master-guide.md**
- Added **Section 9.0: Transition from Validation Spec to Real Spec**
  - What validation passing guarantees
  - Step-by-step creation of real spec from validation
  - What to KEEP, DELETE, ADD
  - Template for real spec header

- Added **Section 13: Quick Start Chat Prompts**
  - 7 ready-to-use prompts for different phases
  - Phase 2 with dual mindset + test mining variant
  - Debugging counterexamples prompt
  - Essential information checklist

---

## Version 1.0 (Initial Release) - January 22, 2026

### Core Documents Created

1. **certora-master-guide.md**
   - Complete 9-phase methodology
   - Project setup templates
   - Phase-by-phase instructions
   - Templates for all documents

2. **certora-workflow.md**
   - Phase overview
   - Visual workflow diagrams
   - Quick reference cards
   - Document purpose mapping

3. **certora-spec-framework.md**
   - CVL 2.0 syntax reference
   - Pattern library
   - Method block patterns
   - Ghost and hook templates

4. **certora-ce-diagnosis-framework.md** (v2.0)
   - Counterexample classification
   - REAL BUG vs SPEC BUG decision tree
   - Causal closure verification
   - Repair patterns

5. **SPEC AUTHORING (CERTORA).md**
   - Deep methodology
   - Non-negotiable axioms
   - Execution closure principles
   - Property classification theory

6. **categorizing-properties.md**
   - Property discovery framework
   - 4 property categories
   - Documentation templates
   - External dependency handling

7. **certora-quickstart-template.md**
   - Quick start guide
   - Copy-paste templates
   - Command cheat sheet

8. **readme.md**
   - Framework overview
   - File descriptions
   - Quick start instructions

---

## Feature Comparison Matrix

| Feature | v1.0 | v1.1 | v1.2 | v1.3 | v1.4 | v1.5 | v2.0 |
|---------|------|------|------|------|------|------|------|
| **Core Methodology** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **9-Phase Workflow** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **CVL Syntax Reference** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅✅ | ✅✅ |
| **CE Diagnosis** | ✅ | ✅ | ✅ | ✅✅ | ✅✅ | ✅✅✅ | ✅✅✅ |
| **Validation Transition** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅✅ |
| **Chat Prompts** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅✅ | ✅✅✅ |
| **Dual Mindset** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Test Mining** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Property Prioritization** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Best Practices Document** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅✅ | ✅✅ |
| **Tutorial Integration** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **5-Step CE Investigation** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Invariant Patterns** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Quick Reference Card** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Advanced CLI Reference** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **CVL Language Deep Dive** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Verification Playbooks** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Vacuous Truth Defense** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Ghost Havocing Diagnosis** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **L/E/NSE Pattern** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Revert/Failure-Path Coverage** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Reachability Validation (satisfy)** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Red Team Hardening** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Validation Evidence Gate** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Document Version Status

| Document | v1.0 | v1.1 | v1.2 | v1.3 | v1.4 | v1.5 | v2.0 | Notes |
|----------|------|------|------|------|------|------|------|-------|
| certora-master-guide.md | 1.0 | 1.1 | 1.1 | 1.3 | 1.3 | 1.5 | 2.0 | §7.5 Evidence Review + §13.4.1 prompt |
| certora-workflow.md | 1.0 | 1.0 | 2.0 | 2.1 | 2.1 | 2.1 | 2.4 | Version bump |
| certora-spec-framework.md | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.1 | 2.2 | Version bump |
| certora-ce-diagnosis-framework.md | 2.0 | 2.0 | 2.0 | 2.1 | 2.1 | 2.2 | 2.3 | Version bump |
| spec-authoring-certora.md | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | Stable — principles |
| categorizing-properties.md | 1.0 | 1.0 | 1.2 | 1.3 | 1.3 | 1.3 | 1.3 | Stable |
| certora-quickstart-template.md | 1.0 | 1.0 | 1.0 | 1.3 | 1.3 | 1.3 | 2.0 | Version bump |
| readme.md | 1.0 | 1.1 | 1.2 | 1.3 | 1.4 | 1.5 | 2.0 | What's New section |
| best-practices-from-certora.md | - | - | - | NEW | 1.0 | 1.1 | 1.1 | Stable |
| quick-reference-v1.3.md | - | - | - | NEW | 1.0 | 1.0 | 1.0 | Cheat sheet |
| tutorial-extraction-summary.md | - | - | - | NEW | 1.0 | 1.0 | 1.0 | Documentation |
| index.md | - | - | - | NEW | 1.0 | 1.0 | 2.0 | Version bump |
| version-history.md | - | - | - | NEW | 1.0 | 1.1 | 2.0 | This entry |
| advanced-cli-reference.md | - | - | - | - | NEW | 1.0 | 1.0 | Stable |
| cvl-language-deep-dive.md | - | - | - | - | - | NEW | 2.0 | Version bump |
| verification-playbooks.md | - | - | - | - | - | NEW | 2.0 | Version bump |
| poc-template-foundry.md | - | - | - | - | NEW | 1.0 | 1.0 | Stable |
| poc-template-hardhat.md | - | - | - | - | NEW | 1.0 | 1.0 | Stable |
| vulnerability-report-template.md | - | - | - | - | NEW | 1.0 | 2.1 | Updated separately |

---

## Migration Guide

### From v1.9 to v2.0

**No breaking changes.** All v1.9 features remain compatible.

**To adopt v2.0 enhancements:**

1. **After validation ALL PASS, complete Validation Evidence Review:**
   - Follow `certora-master-guide.md` Section 7.5 (new)
   - Inspect satisfy witnesses for non-degeneracy (amount > 0, state changes)
   - Inspect ghost sync witnesses for non-triviality (ghost ≠ 0, hook fires)
   - Cross-reference mutation path whitelists against Phase 0 entry points
   - Re-run with `"rule_sanity": "advanced"` at least once
   - Fill in the evidence template in `causal_validation.md`
   - Sign off before proceeding to Phase 7

2. **Use the new chat prompt for evidence review:**
   - Section 13.4.1 provides a ready-to-use prompt for AI-assisted evidence review
   - Copy and use it after certoraRun shows ALL PASS

3. **Update Phase 6 Sanity Gate:**
   - 6 new checklist items under "Validation Evidence Review" section
   - Must be checked before entering Phase 7

### From v1.4 to v1.5

**No breaking changes.** All v1.4 features remain compatible.

**To adopt v1.5 enhancements:**

1. **Use CVL Language Deep Dive for spec writing:**
   - Reference `cvl-language-deep-dive.md` for type system, ghost variables, hooks, invariants
   - Contains 20 detailed sections covering all CVL 2.0 concepts
   - Use the Quick Reference Tables (Section 20) for at-a-glance lookup

2. **Use Verification Playbooks for worked examples:**
   - Use `verification-playbooks.md` for production-ready spec patterns
   - ERC-20 Playbook: 22 rules across 4 phases (start here for token verification)
   - WETH Playbook: Solvency invariants with persistent ghost + CALL hook
   - ERC-721 Playbook: NFT verification with DISPATCHER for callbacks

3. **Apply vacuous truth defense:**
   - Check `best-practices-from-certora.md` Section 7
   - Always pair `require` with `satisfy` to prove witness existence
   - Use `assert ... => ...` instead of `require ...; assert ...` when possible

4. **Adopt requireInvariant lifecycle:**
   - Check `best-practices-from-certora.md` Section 8
   - Prove invariant independently → then use `requireInvariant` in rules

5. **Handle self-transfer edge cases:**
   - Check `best-practices-from-certora.md` Section 9
   - Always separate `from == to` and `from != to` cases in token transfer rules

6. **Debug ghost havocing:**
   - Check `certora-ce-diagnosis-framework.md` ghost havocing section
   - Use `persistent ghost` + `hook CALL` pattern for native balance tracking

### From v1.3 to v1.4

**No breaking changes.** All v1.3 features remain compatible.

**To adopt v1.4 enhancements:**

1. **Use Advanced CLI Reference for performance:**
   - Reference `advanced-cli-reference.md` for timeout optimization
   - Use `--smt_timeout`, `--loop_iter`, `--optimistic_loop` flags
   - Apply modular verification with `--rule` flag for large specs

2. **Use PoC templates for findings:**
   - Use `POC_TEMPLATE_Foundry.md` or `POC_TEMPLATE_HARDHAT.md` to translate Certora counterexamples into executable PoCs
   - Use `VULNERABILITY_REPORT_TEMPLATE.md` for formal write-ups

### From v1.2 to v1.3

**No breaking changes.** All v1.2 features remain compatible.

**To adopt v1.3 enhancements:**

1. **Add prioritization to existing properties:**
   - Open your `candidate_properties.md`
   - Add **Priority:** field (HIGH/MEDIUM/LOW) to each property
   - Reference `categorizing-properties.md` Section 7 for criteria

2. **Reference best practices during verification:**
   - Use `best-practices-from-certora.md` Section 1 during Phase 2
   - Use Section 2 for CE debugging
   - Use Section 3 for invariant design

3. **Use quick reference card:**
   - Print or keep open `quick-reference-v1.3.md`
   - Follow pre-verification checklist before running prover

4. **Apply 5-step CE investigation:**
   - When debugging, follow the systematic workflow
   - Break down complex expressions
   - Document findings using bug template

### From v1.1 to v1.2

**No breaking changes.**

**To adopt v1.2 enhancements:**
- Apply dual mindset when discovering properties
- Mine existing tests for implicit properties
- Use updated chat prompt (Section 13.3.1)

### From v1.0 to v1.1

**No breaking changes.**

**To adopt v1.1 enhancements:**
- Use validation-to-real-spec transition guide (Section 9.0)
- Use chat prompts for phase-specific guidance (Section 13)

---

## Roadmap

### Completed
- **v1.4** — Advanced CLI Reference, PoC templates, Vulnerability Report template
- **v1.5** — RareSkills Certora Book integration, CVL Language Deep Dive, Verification Playbooks (ERC-20/WETH/ERC-721), vacuous truth defense, requireInvariant lifecycle, ghost havocing diagnosis, Liveness/Effect/No-Side-Effect pattern
- **v1.6** — Revert/failure-path coverage (@withrevert, biconditional, MUST REVERT WHEN)
- **v1.7** — Prover v8.8.0 builtin rules (uncheckedOverflow, safeCasting)
- **v1.8** — Reachability validation (satisfy in validation spec)
- **v1.9** — Red Team Hardening (failure-path satisfy, custom summary accuracy, invariant DAG)
- **v2.0** — Validation Evidence Gate (witness inspection, advanced sanity, evidence sign-off)

### Planned
- Integration examples from real audit engagements
- Property library expansion (governance, vaults, staking)
- Automated property extraction tooling
- Multi-contract verification patterns

### Under Consideration
- Cheatsheet for common HAVOC scenarios
- Troubleshooting decision trees
- Integration with security tooling
- Community property database

---

## Contributors

Framework developed by the Certora community, enhanced with official Certora tutorial best practices.

Tutorial sources:
- Certora Tutorials Repository (https://github.com/Certora/Tutorials)
- Auction Demonstration examples
- Official Certora documentation

---

## Support

For questions or issues:
1. Check `best-practices-from-certora.md` Section 6 (Common Pitfalls)
2. Review `certora-ce-diagnosis-framework.md` for debugging
3. Consult `quick-reference-v1.3.md` for quick answers
4. Reference `cvl-language-deep-dive.md` for CVL language details
5. Use `verification-playbooks.md` for worked examples
6. Reference specific tutorial lessons via `tutorial-extraction-summary.md`

---

**Current Stable Version:** 2.0  
**Status:** Production-Ready  
**Last Updated:** February 2026
