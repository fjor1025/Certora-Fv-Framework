# Framework Index & Navigation Guide

> **Quick navigation for the Certora Formal Verification Framework v1.5**

---

## Core Learning Path (Start Here)

**If you're new to this framework:**

1. **READ FIRST:** [README.md](README.md) - Framework overview and what's new
2. **METHODOLOGY:** [SPEC AUTHORING (CERTORA).md](SPEC%20AUTHORING%20%28CERTORA%29.md) - Understand the WHY
3. **WORKFLOW:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) - Complete step-by-step process
4. **START VERIFICATION:** [CERTORA_QUICKSTART_TEMPLATE.md](CERTORA_QUICKSTART_TEMPLATE.md) - Copy and fill

---

## Document Reference by Purpose

### Planning & Setup
| Document | Use When |
|----------|----------|
| [README.md](README.md) | Understanding framework capabilities |
| [CERTORA_QUICKSTART_TEMPLATE.md](CERTORA_QUICKSTART_TEMPLATE.md) | Setting up new verification project |
| [CERTORA_WORKFLOW.md](CERTORA_WORKFLOW.md) | Need visual workflow overview |

### Analysis & Discovery (Phases 0-6)
| Document | Use When |
|----------|----------|
| [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) | Following complete methodology |
| [Categorizing_Properties.md](Categorizing_Properties.md) | Discovering security properties (Phase 2) |
| [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) | Learning proven techniques |
| [SPEC AUTHORING (CERTORA).md](SPEC%20AUTHORING%20%28CERTORA%29.md) | Deep understanding of principles |

### Writing Specs (Phase 7)
| Document | Use When |
|----------|----------|
| [CERTORA_SPEC_FRAMEWORK.md](CERTORA_SPEC_FRAMEWORK.md) | Writing CVL code |
| [CVL_LANGUAGE_DEEP_DIVE.md](CVL_LANGUAGE_DEEP_DIVE.md) | CVL type system, ghosts, hooks, invariants ⭐ **NEW v1.5** |
| [VERIFICATION_PLAYBOOKS.md](VERIFICATION_PLAYBOOKS.md) | Copy-paste worked examples (ERC-20, WETH, ERC-721) ⭐ **NEW v1.5** |
| [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) | Invariant patterns & harness design |
| [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) | Quick syntax lookup |

### Debugging (When Rules Fail)
| Document | Use When |
|----------|----------|
| [CERTORA_CE_DIAGNOSIS_FRAMEWORK.md](CERTORA_CE_DIAGNOSIS_FRAMEWORK.md) | Investigating counterexamples |
| [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) | Section 2: 5-step investigation |
| [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) | Quick troubleshooting |

### Performance & CLI
| Document | Use When |
|----------|----------|
| [ADVANCED_CLI_REFERENCE.md](ADVANCED_CLI_REFERENCE.md) | Timeout optimization, advanced flags ⭐ **NEW v1.4** |

### Reference & Learning
| Document | Use When |
|----------|----------|
| [TUTORIAL_EXTRACTION_SUMMARY.md](TUTORIAL_EXTRACTION_SUMMARY.md) | Understanding tutorial integration |
| [VERSION_HISTORY.md](VERSION_HISTORY.md) | Checking what's new, migration guides |
| [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) | Need printable cheat sheet |

---

## Quick Access by Phase

### Phase 0: Contract Analysis
**Primary:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 3  
**Reference:** [SPEC AUTHORING (CERTORA).md](SPEC%20AUTHORING%20%28CERTORA%29.md) Phase 0  
**Checklist:** Enumerate entry points, storage, external calls

### Phase -1: Execution Closure
**Primary:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 4  
**Reference:** [SPEC AUTHORING (CERTORA).md](SPEC%20AUTHORING%20%28CERTORA%29.md) Phase -1  
**Checklist:** Complete interaction ownership table

### Phase 2: Property Discovery
**Primary:** [Categorizing_Properties.md](Categorizing_Properties.md)  
**Enhanced:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 1  
**Tips:** Avoid 4 fatal mistakes, use dual mindset, mine tests  
**Checklist:** All properties in plain English + prioritized

### Phase 2.5: Classification (Invariant vs Rule)
**Primary:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 6  
**Reference:** [CERTORA_WORKFLOW.md](CERTORA_WORKFLOW.md) Step 4  
**Checklist:** Each property classified with reasoning

### Phase 3.5: Causal Validation
**Primary:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 7  
**Reference:** [CERTORA_WORKFLOW.md](CERTORA_WORKFLOW.md) Step 5  
**Checklist:** Validation spec PASSED

### Phase 4-6: Modeling & Sanity
**Primary:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 8  
**Patterns:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 3 & 4  
**Checklist:** Sanity gate all checked

### Phase 7: Write CVL
**Primary:** [CERTORA_SPEC_FRAMEWORK.md](CERTORA_SPEC_FRAMEWORK.md)  
**CVL Reference:** [CVL_LANGUAGE_DEEP_DIVE.md](CVL_LANGUAGE_DEEP_DIVE.md) ⭐ **NEW v1.5**  
**Worked Examples:** [VERIFICATION_PLAYBOOKS.md](VERIFICATION_PLAYBOOKS.md) ⭐ **NEW v1.5**  
**Guide:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 9  
**Patterns:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Sections 3-5  
**Checklist:** Pre-verification checklist in QUICK_REFERENCE

### Debugging: Counterexamples
**Primary:** [CERTORA_CE_DIAGNOSIS_FRAMEWORK.md](CERTORA_CE_DIAGNOSIS_FRAMEWORK.md)  
**Workflow:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 2  
**Tips:** 5-step investigation, break down complex expressions

---

## Find Information By Topic

### Property Discovery
- **Main Guide:** [Categorizing_Properties.md](Categorizing_Properties.md)
- **Techniques:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 1
- **Fatal Mistakes:** BEST_PRACTICES Section 1.1
- **Prioritization:** Categorizing_Properties Section 7

### Invariant Design
- **Patterns:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 3
- **Monotonicity:** BEST_PRACTICES Section 3.2
- **Conservation Laws:** BEST_PRACTICES Section 3.2
- **Preserved Blocks:** BEST_PRACTICES Section 3.3

### Harness Creation
- **Best Practices:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 4
- **Templates:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 11.1
- **When to Use:** BEST_PRACTICES Section 4.1

### Loop Handling
- **Strategy:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 5
- **Flags:** BEST_PRACTICES Section 5.1
- **Configuration:** CERTORA_MASTER_GUIDE Section 9.2

### Ghost Variables
- **Design:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 8.2
- **Patterns:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 3.2
- **Hooks:** CERTORA_SPEC_FRAMEWORK hooks section

### CVL Syntax
- **Complete Reference:** [CVL_LANGUAGE_DEEP_DIVE.md](CVL_LANGUAGE_DEEP_DIVE.md) ⭐ **NEW v1.5** (20 sections)
- **Templates & Patterns:** [CERTORA_SPEC_FRAMEWORK.md](CERTORA_SPEC_FRAMEWORK.md)
- **Quick Lookup:** [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) Section 12.2
- **Worked Examples:** [VERIFICATION_PLAYBOOKS.md](VERIFICATION_PLAYBOOKS.md) ⭐ **NEW v1.5**

### Counterexample Debugging
- **Framework:** [CERTORA_CE_DIAGNOSIS_FRAMEWORK.md](CERTORA_CE_DIAGNOSIS_FRAMEWORK.md)
- **5-Step Process:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 2
- **Call Trace Analysis:** BEST_PRACTICES Section 2.2

### Common Issues
- **Pitfalls:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 6
- **Troubleshooting:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 12.4
- **Compilation Errors:** CERTORA_MASTER_GUIDE Section 10.2

---

## Learning Resources

### Official Certora Tutorials
The framework integrates techniques from:
- **Lesson 02:** Investigate Violations → CE debugging workflow
- **Lesson 06:** Thinking Properties → Property discovery
- **Lessons 07-08:** Invariants → Inductive reasoning & preserved blocks
- **Lesson 11:** Loops → Loop handling strategies
- **Lesson 15:** Harness → Harness design patterns
- **Auction Demo:** Property prioritization methodology

**See:** [TUTORIAL_EXTRACTION_SUMMARY.md](TUTORIAL_EXTRACTION_SUMMARY.md) for complete mapping

### Framework Evolution
- **v1.0 → v1.5:** [VERSION_HISTORY.md](VERSION_HISTORY.md)
- **Migration Guides:** VERSION_HISTORY migration sections
- **Feature Comparison:** VERSION_HISTORY feature matrix

---

## Verification Workflow Checklist

**Print this and check off as you progress:**

- [ ] **Setup** - Created folder structure (QUICKSTART Section 1)
- [ ] **Phase 0** - Contract analysis complete (MASTER_GUIDE Section 3)
- [ ] **Phase -1** - Execution closure mapped (MASTER_GUIDE Section 4)
- [ ] **Phase 2** - Properties discovered & prioritized (Categorizing_Properties)
- [ ] **Phase 2.5** - Properties classified (MASTER_GUIDE Section 6)
- [ ] **Phase 3.5** - Validation spec PASSED (MASTER_GUIDE Section 7)
- [ ] **Phase 4-6** - Modeling complete, sanity checks passed (MASTER_GUIDE Section 8)
- [ ] **Phase 7** - Real spec written (MASTER_GUIDE Section 9)
- [ ] **Run** - Prover executed (MASTER_GUIDE Section 10)
- [ ] **Debug** - Counterexamples investigated (CE_DIAGNOSIS_FRAMEWORK)
- [ ] **Done** - All properties verified ✅

---

## Templates & Examples

### Copy-Paste Templates
- **Project Setup:** [CERTORA_QUICKSTART_TEMPLATE.md](CERTORA_QUICKSTART_TEMPLATE.md) Section 1
- **Property Template:** [Categorizing_Properties.md](Categorizing_Properties.md) Property Documentation Template
- **Prioritization Template:** Categorizing_Properties Section 7.2
- **CVL Spec Structure:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 9.1
- **Harness Template:** CERTORA_MASTER_GUIDE Section 11.1
- **Bug Report Template:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 2.3

### Pattern Library
- **Invariant Patterns:** BEST_PRACTICES Section 3.2
- **Ghost Patterns:** CERTORA_MASTER_GUIDE Section 8.2
- **Helper Functions:** CERTORA_SPEC_FRAMEWORK helper section

---

## Tools & Commands

### Essential Commands
```bash
# Clear cache
rm -rf .certora_internal

# Run validation (always first)
certoraRun certora/confs/validation_contract.conf

# Run real spec
certoraRun certora/confs/Contract.conf

# Run single rule
certoraRun certora/confs/Contract.conf --rule "ruleName"
```

**Full Reference:** [QUICK_REFERENCE_v1.3.md](QUICK_REFERENCE_v1.3.md) or MASTER_GUIDE Section 10.1

---

## Quick Tips

### Before You Start
1. ✅ Have all framework files in your project root
2. ✅ Read SPEC AUTHORING for philosophy
3. ✅ Understand the 9-phase workflow
4. ✅ Print QUICK_REFERENCE cheat sheet

### During Verification
1. ✅ Never skip causal validation (Phase 3.5)
2. ✅ Prioritize properties (HIGH first)
3. ✅ Reference BEST_PRACTICES for patterns
4. ✅ Break down complex expressions

### When Debugging
1. ✅ Use 5-step investigation process
2. ✅ Check for HAVOC in call trace
3. ✅ Verify causal closure first
4. ✅ Document all decisions

---

## Support & Help

### Stuck? Check These First:
1. **Common Pitfalls:** [BEST_PRACTICES_FROM_CERTORA.md](BEST_PRACTICES_FROM_CERTORA.md) Section 6
2. **Troubleshooting:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md) Section 12.4
3. **CE Debugging:** [CERTORA_CE_DIAGNOSIS_FRAMEWORK.md](CERTORA_CE_DIAGNOSIS_FRAMEWORK.md)

### Chat Assistant Prompts
Use MASTER_GUIDE Section 13 for ready-to-use prompts for each phase.

---

## Framework Files Summary

Total: **19 documents** (v1.5)

**Core (8):**
1. README.md
2. CERTORA_MASTER_GUIDE.md
3. CERTORA_WORKFLOW.md
4. CERTORA_SPEC_FRAMEWORK.md
5. CERTORA_CE_DIAGNOSIS_FRAMEWORK.md
6. SPEC AUTHORING (CERTORA).md
7. Categorizing_Properties.md
8. CERTORA_QUICKSTART_TEMPLATE.md

**Enhanced (3 — added v1.3):**
9. BEST_PRACTICES_FROM_CERTORA.md
10. QUICK_REFERENCE_v1.3.md
11. TUTORIAL_EXTRACTION_SUMMARY.md

**v1.4 Additions (4):**
12. ADVANCED_CLI_REFERENCE.md ⭐ CLI & performance
13. POC_TEMPLATE_Foundry.md ⭐ Foundry PoC template
14. POC_TEMPLATE_HARDHAT.md ⭐ Hardhat PoC template
15. VULNERABILITY_REPORT_TEMPLATE.md ⭐ Report template

**v1.5 Additions (2):**
16. CVL_LANGUAGE_DEEP_DIVE.md ⭐ Complete CVL language reference
17. VERIFICATION_PLAYBOOKS.md ⭐ ERC-20/WETH/ERC-721 worked examples

**Meta (2):**
18. VERSION_HISTORY.md
19. INDEX.md (this file)

---

**Framework Version:** 1.5 (RareSkills Integration)  
**Status:** Production-Ready  
**Last Updated:** February 8, 2026

**Start Your Verification Journey:** [CERTORA_MASTER_GUIDE.md](CERTORA_MASTER_GUIDE.md)
