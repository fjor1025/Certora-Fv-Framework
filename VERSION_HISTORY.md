# Framework Version History

> **Current Version:** 1.5 (RareSkills Integration)  
> **Last Updated:** February 2026

---

## Version 1.5 (RareSkills Integration) - February 2026

### Knowledge Source
Complete analysis of the **RareSkills Certora Book** — 35 chapters, 60,000+ words, produced in collaboration with Certora. Every chapter was systematically read and knowledge extracted to fill 19 identified gaps in the framework.

### Major Additions

#### New Documents

1. **CVL_LANGUAGE_DEEP_DIVE.md** — Complete CVL language reference (20 sections)
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

2. **VERIFICATION_PLAYBOOKS.md** — Complete worked verification examples
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

3. **CERTORA_SPEC_FRAMEWORK.md**
   - Added `definition` blocks section (nonpayable, nonzerosender, balanceLimited)
   - Added Liveness/Effect/No-Side-Effect rule template in CVL 2.0 Templates section

4. **BEST_PRACTICES_FROM_CERTORA.md**
   - Added Section 7: Vacuous Truth & Tautology Defense
   - Added Section 8: The `require` → `requireInvariant` Lifecycle
   - Added Section 9: Self-Transfer & Edge Case Patterns
   - Updated framework phase mapping table
   - Expanded pre-run checklist with v1.5 items

5. **CERTORA_CE_DIAGNOSIS_FRAMEWORK.md**
   - Added Ghost Havocing Diagnosis Guide with symptoms, steps, and fixes
   - Added Persistent Ghost + CALL Hook template
   - Added havocing behavior comparison table (regular vs persistent)

6. **CERTORA_MASTER_GUIDE.md**
   - Version bump to v1.5
   - Added CVL_LANGUAGE_DEEP_DIVE.md and VERIFICATION_PLAYBOOKS.md to document table

7. **README.md**
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
1. **ADVANCED_CLI_REFERENCE.md** - Comprehensive performance optimization and advanced CLI guide
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

**QUICK_REFERENCE_v1.3.md**
- Added **"⚡ Performance & Advanced Flags"** section
  - Timeout mitigation quick reference table
  - Advanced debugging flags table
  - Quick performance commands
- Updated document links to include ADVANCED_CLI_REFERENCE
- Added loop/array and multi-version project references

**CERTORA_MASTER_GUIDE.md**
- Updated to v1.4
- Added ADVANCED_CLI_REFERENCE to framework documents table (Section 1.1)
- Added **Section 10.4: Performance Optimization & Timeout Mitigation**
  - Quick timeout fixes table
  - Common performance commands
  - Performance decision tree
  - Advanced debugging flags
  - Direct reference to detailed strategies in ADVANCED_CLI_REFERENCE

**README.md**
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
1. **BEST_PRACTICES_FROM_CERTORA.md** - Comprehensive extraction from official Certora tutorials
   - Property discovery techniques (Lesson 06)
   - 5-step CE investigation workflow (Lesson 02)
   - Invariant design patterns (Lessons 07, 08, 15)
   - Harness best practices (Lesson 15)
   - Loop handling strategies (Lesson 11)
   - Common pitfalls and anti-patterns

2. **QUICK_REFERENCE_v1.3.md** - Printable cheat sheet
   - Phase checklists with best practices
   - Invariant pattern quick reference
   - Pre-verification checklist
   - Command cheat sheet

3. **TUTORIAL_EXTRACTION_SUMMARY.md** - Documentation of extraction process
   - Tutorial-to-framework mapping
   - Key insights extracted
   - Integration points
   - Usage guidelines

#### Enhanced Documents

**Categorizing_Properties.md**
- Added **Section 7: Property Prioritization**
  - HIGH / MEDIUM / LOW priority levels
  - Impact-based prioritization matrix
  - Priority assignment methodology
  - Verification time budget allocation
  - Complete property entry template
- Updated checklist to v1.3

**CERTORA_CE_DIAGNOSIS_FRAMEWORK.md**
- Updated to v2.1
- Added **Tutorial-Based Investigation Workflow** section
  - 5-step systematic investigation process
  - Call trace analysis checklist (storage, arguments, returns)
  - Expression breakdown best practices
  - Bug documentation template

**CERTORA_MASTER_GUIDE.md**
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

**CERTORA_WORKFLOW.md**
- Updated to v2.1
- Enhanced framework overview diagram with BEST_PRACTICES and updated Categorizing_Properties
- Added v1.3 enhancements note to Phase 2
- Added reference to 5-step CE debugging

**CERTORA_QUICKSTART_TEMPLATE.md**
- Added "What's New in v1.3" section
- References to prioritization, dual mindset, test mining

**README.md**
- Updated to v1.3
- Added comprehensive "What's New" section
- Added BEST_PRACTICES to file list
- Previous enhancements section updated

---

## Version 1.2 (Dual Mindset Enhanced) - January 31, 2026

### Enhancements

**Categorizing_Properties.md**
- Added **Section 5: RIGHT / WRONG BEHAVIOR DUAL CHECKLIST**
  - Dual mindset approach: "Should Always" vs "Should Never"
  - Attack vector enumeration
  - Comprehensive threat modeling
  
- Added **Section 6: Mining Properties from Test Suites**
  - Test-to-property translation patterns
  - Coverage blind spot identification
  - Implicit invariant extraction

**CERTORA_MASTER_GUIDE.md**
- Updated Phase 2 chat prompt with dual mindset option (Section 13.3.1)

**README.md**
- Added v1.2 features section

---

## Version 1.1 (Transition Enhanced) - January 27, 2026

### Enhancements

**CERTORA_MASTER_GUIDE.md**
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

1. **CERTORA_MASTER_GUIDE.md**
   - Complete 9-phase methodology
   - Project setup templates
   - Phase-by-phase instructions
   - Templates for all documents

2. **CERTORA_WORKFLOW.md**
   - Phase overview
   - Visual workflow diagrams
   - Quick reference cards
   - Document purpose mapping

3. **CERTORA_SPEC_FRAMEWORK.md**
   - CVL 2.0 syntax reference
   - Pattern library
   - Method block patterns
   - Ghost and hook templates

4. **CERTORA_CE_DIAGNOSIS_FRAMEWORK.md** (v2.0)
   - Counterexample classification
   - REAL BUG vs SPEC BUG decision tree
   - Causal closure verification
   - Repair patterns

5. **SPEC AUTHORING (CERTORA).md**
   - Deep methodology
   - Non-negotiable axioms
   - Execution closure principles
   - Property classification theory

6. **Categorizing_Properties.md**
   - Property discovery framework
   - 4 property categories
   - Documentation templates
   - External dependency handling

7. **CERTORA_QUICKSTART_TEMPLATE.md**
   - Quick start guide
   - Copy-paste templates
   - Command cheat sheet

8. **README.md**
   - Framework overview
   - File descriptions
   - Quick start instructions

---

## Feature Comparison Matrix

| Feature | v1.0 | v1.1 | v1.2 | v1.3 | v1.4 | v1.5 |
|---------|------|------|------|------|------|------|
| **Core Methodology** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **9-Phase Workflow** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **CVL Syntax Reference** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅✅ |
| **CE Diagnosis** | ✅ | ✅ | ✅ | ✅✅ | ✅✅ | ✅✅✅ |
| **Validation Transition** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Chat Prompts** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅✅ |
| **Dual Mindset** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Test Mining** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Property Prioritization** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Best Practices Document** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅✅ |
| **Tutorial Integration** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **5-Step CE Investigation** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Invariant Patterns** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Quick Reference Card** | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Advanced CLI Reference** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **CVL Language Deep Dive** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Verification Playbooks** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Vacuous Truth Defense** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Ghost Havocing Diagnosis** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **L/E/NSE Pattern** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Document Version Status

| Document | v1.0 | v1.1 | v1.2 | v1.3 | v1.4 | v1.5 | Notes |
|----------|------|------|------|------|------|------|-------|
| CERTORA_MASTER_GUIDE.md | 1.0 | 1.1 | 1.1 | 1.3 | 1.3 | 1.5 | Added v1.5 doc refs + Section 13 prompts |
| CERTORA_WORKFLOW.md | 1.0 | 1.0 | 2.0 | 2.1 | 2.1 | 2.1 | Stable |
| CERTORA_SPEC_FRAMEWORK.md | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.1 | Added definitions + L/E/NSE template |
| CERTORA_CE_DIAGNOSIS_FRAMEWORK.md | 2.0 | 2.0 | 2.0 | 2.1 | 2.1 | 2.2 | Added ghost havocing diagnosis |
| SPEC AUTHORING (CERTORA).md | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | Stable - principles |
| Categorizing_Properties.md | 1.0 | 1.0 | 1.2 | 1.3 | 1.3 | 1.3 | Stable |
| CERTORA_QUICKSTART_TEMPLATE.md | 1.0 | 1.0 | 1.0 | 1.3 | 1.3 | 1.3 | Stable |
| README.md | 1.0 | 1.1 | 1.2 | 1.3 | 1.4 | 1.5 | Version tracking |
| BEST_PRACTICES_FROM_CERTORA.md | - | - | - | NEW | 1.0 | 1.1 | Added Sections 7-9 |
| QUICK_REFERENCE_v1.3.md | - | - | - | NEW | 1.0 | 1.0 | Cheat sheet |
| TUTORIAL_EXTRACTION_SUMMARY.md | - | - | - | NEW | 1.0 | 1.0 | Documentation |
| INDEX.md | - | - | - | NEW | 1.0 | 1.0 | Navigation |
| VERSION_HISTORY.md | - | - | - | NEW | 1.0 | 1.1 | Updated matrices + migration |
| ADVANCED_CLI_REFERENCE.md | - | - | - | - | NEW | 1.0 | Performance & CLI |
| CVL_LANGUAGE_DEEP_DIVE.md | - | - | - | - | - | NEW | Complete CVL reference |
| VERIFICATION_PLAYBOOKS.md | - | - | - | - | - | NEW | ERC-20/WETH/ERC-721 worked examples |
| POC_TEMPLATE_Foundry.md | - | - | - | - | NEW | 1.0 | Foundry PoC template |
| POC_TEMPLATE_HARDHAT.md | - | - | - | - | NEW | 1.0 | Hardhat PoC template |
| VULNERABILITY_REPORT_TEMPLATE.md | - | - | - | - | NEW | 1.0 | Vulnerability report template |

---

## Migration Guide

### From v1.4 to v1.5

**No breaking changes.** All v1.4 features remain compatible.

**To adopt v1.5 enhancements:**

1. **Use CVL Language Deep Dive for spec writing:**
   - Reference `CVL_LANGUAGE_DEEP_DIVE.md` for type system, ghost variables, hooks, invariants
   - Contains 20 detailed sections covering all CVL 2.0 concepts
   - Use the Quick Reference Tables (Section 20) for at-a-glance lookup

2. **Use Verification Playbooks for worked examples:**
   - Use `VERIFICATION_PLAYBOOKS.md` for production-ready spec patterns
   - ERC-20 Playbook: 22 rules across 4 phases (start here for token verification)
   - WETH Playbook: Solvency invariants with persistent ghost + CALL hook
   - ERC-721 Playbook: NFT verification with DISPATCHER for callbacks

3. **Apply vacuous truth defense:**
   - Check `BEST_PRACTICES_FROM_CERTORA.md` Section 7
   - Always pair `require` with `satisfy` to prove witness existence
   - Use `assert ... => ...` instead of `require ...; assert ...` when possible

4. **Adopt requireInvariant lifecycle:**
   - Check `BEST_PRACTICES_FROM_CERTORA.md` Section 8
   - Prove invariant independently → then use `requireInvariant` in rules

5. **Handle self-transfer edge cases:**
   - Check `BEST_PRACTICES_FROM_CERTORA.md` Section 9
   - Always separate `from == to` and `from != to` cases in token transfer rules

6. **Debug ghost havocing:**
   - Check `CERTORA_CE_DIAGNOSIS_FRAMEWORK.md` ghost havocing section
   - Use `persistent ghost` + `hook CALL` pattern for native balance tracking

### From v1.3 to v1.4

**No breaking changes.** All v1.3 features remain compatible.

**To adopt v1.4 enhancements:**

1. **Use Advanced CLI Reference for performance:**
   - Reference `ADVANCED_CLI_REFERENCE.md` for timeout optimization
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
   - Reference `Categorizing_Properties.md` Section 7 for criteria

2. **Reference best practices during verification:**
   - Use `BEST_PRACTICES_FROM_CERTORA.md` Section 1 during Phase 2
   - Use Section 2 for CE debugging
   - Use Section 3 for invariant design

3. **Use quick reference card:**
   - Print or keep open `QUICK_REFERENCE_v1.3.md`
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

### Planned for v1.6
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
1. Check `BEST_PRACTICES_FROM_CERTORA.md` Section 6 (Common Pitfalls)
2. Review `CERTORA_CE_DIAGNOSIS_FRAMEWORK.md` for debugging
3. Consult `QUICK_REFERENCE_v1.3.md` for quick answers
4. Reference `CVL_LANGUAGE_DEEP_DIVE.md` for CVL language details
5. Use `VERIFICATION_PLAYBOOKS.md` for worked examples
6. Reference specific tutorial lessons via `TUTORIAL_EXTRACTION_SUMMARY.md`

---

**Current Stable Version:** 1.5  
**Status:** Production-Ready  
**Last Updated:** February 8, 2026
