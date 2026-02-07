# Framework Version History

> **Current Version:** 1.4 (Performance-Enhanced + Advanced CLI)  
> **Last Updated:** February 7, 2026

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

| Feature | v1.0 | v1.1 | v1.2 | v1.3 |
|---------|------|------|------|------|
| **Core Methodology** | ✅ | ✅ | ✅ | ✅ |
| **9-Phase Workflow** | ✅ | ✅ | ✅ | ✅ |
| **CVL Syntax Reference** | ✅ | ✅ | ✅ | ✅ |
| **CE Diagnosis** | ✅ | ✅ | ✅ | ✅✅ |
| **Validation Transition** | ❌ | ✅ | ✅ | ✅ |
| **Chat Prompts** | ❌ | ✅ | ✅ | ✅ |
| **Dual Mindset** | ❌ | ❌ | ✅ | ✅ |
| **Test Mining** | ❌ | ❌ | ✅ | ✅ |
| **Property Prioritization** | ❌ | ❌ | ❌ | ✅ |
| **Best Practices Document** | ❌ | ❌ | ❌ | ✅ |
| **Tutorial Integration** | ❌ | ❌ | ❌ | ✅ |
| **5-Step CE Investigation** | ❌ | ❌ | ❌ | ✅ |
| **Invariant Patterns** | ❌ | ❌ | ❌ | ✅ |
| **Quick Reference Card** | ❌ | ❌ | ❌ | ✅ |

---

## Document Version Status

| Document | v1.0 | v1.1 | v1.2 | v1.3 | Notes |
|----------|------|------|------|------|-------|
| CERTORA_MASTER_GUIDE.md | 1.0 | 1.1 | 1.1 | 1.3 | Added prioritization refs |
| CERTORA_WORKFLOW.md | 1.0 | 1.0 | 2.0 | 2.1 | Enhanced diagrams |
| CERTORA_SPEC_FRAMEWORK.md | 1.0 | 1.0 | 1.0 | 1.0 | Stable - CVL syntax |
| CERTORA_CE_DIAGNOSIS_FRAMEWORK.md | 2.0 | 2.0 | 2.0 | 2.1 | Added tutorial workflow |
| SPEC AUTHORING (CERTORA).md | 1.0 | 1.0 | 1.0 | 1.0 | Stable - principles |
| Categorizing_Properties.md | 1.0 | 1.0 | 1.2 | 1.3 | Added prioritization |
| CERTORA_QUICKSTART_TEMPLATE.md | 1.0 | 1.0 | 1.0 | 1.3 | Added v1.3 overview |
| README.md | 1.0 | 1.1 | 1.2 | 1.3 | Version tracking |
| BEST_PRACTICES_FROM_CERTORA.md | - | - | - | NEW | Tutorial extraction |
| QUICK_REFERENCE_v1.3.md | - | - | - | NEW | Cheat sheet |
| TUTORIAL_EXTRACTION_SUMMARY.md | - | - | - | NEW | Documentation |

---

## Migration Guide

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

### Planned for v1.4
- Integration examples from real audits
- Property library for common patterns (ERC20, governance, vaults)
- Automated property extraction tools
- Video tutorial links

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
4. Reference specific tutorial lessons via `TUTORIAL_EXTRACTION_SUMMARY.md`

---

**Current Stable Version:** 1.3  
**Status:** Production-Ready  
**Last Tested:** February 5, 2026
