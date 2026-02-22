# Framework Index & Navigation Guide

> **Quick navigation for the Certora Formal Verification Framework v3.0**

---

## Core Learning Path (Start Here)

**If you're new to this framework:**

1. **READ FIRST:** [readme.md](readme.md) - Framework overview and mission
2. **METHODOLOGY:** [SPEC AUTHORING (CERTORA).md](spec-authoring-certora.md) - Understand the WHY
3. **WORKFLOW:** [certora-master-guide.md](certora-master-guide.md) - Complete step-by-step process
4. **START VERIFICATION:** [certora-quickstart-template.md](certora-quickstart-template.md) - Copy and fill
5. **OFFENSIVE MODE:** [impact-spec-template.md](impact-spec-template.md) - Hunt for exploits ⭐ NEW v3.0

---

## Document Reference by Purpose

### Planning & Setup
| Document | Use When |
|----------|----------|
| [readme.md](readme.md) | Understanding framework capabilities |
| [certora-quickstart-template.md](certora-quickstart-template.md) | Setting up new verification project |
| [certora-workflow.md](certora-workflow.md) | Need visual workflow overview |

### Analysis & Discovery (Phases 0-6)
| Document | Use When |
|----------|----------|
| [certora-master-guide.md](certora-master-guide.md) | Following complete methodology |
| [categorizing-properties.md](categorizing-properties.md) | Discovering security properties (Phase 2) |
| [best-practices-from-certora.md](best-practices-from-certora.md) | Learning proven techniques |
| [SPEC AUTHORING (CERTORA).md](spec-authoring-certora.md) | Deep understanding of principles |

### Writing Specs (Phase 7)
| Document | Use When |
|----------|----------|
| [certora-spec-framework.md](certora-spec-framework.md) | Writing CVL code |
| [cvl-language-deep-dive.md](cvl-language-deep-dive.md) | CVL type system, ghosts, hooks, invariants, builtin rules §19.1 ⭐ |
| [verification-playbooks.md](verification-playbooks.md) | Copy-paste worked examples (ERC-20, WETH, ERC-721) ⭐ |
| [best-practices-from-certora.md](best-practices-from-certora.md) | Invariant patterns & harness design |
| [quick-reference-v1.3.md](quick-reference-v1.3.md) | Quick syntax lookup |

### Offensive Verification (Phase 8) ⭐ NEW v3.0
| Document | Use When |
|----------|----------|
| [impact-spec-template.md](impact-spec-template.md) | Economic impact tracking, anti-invariants |
| [multi-step-attacks-template.md](multi-step-attacks-template.md) | Flash loan, sandwich, staged attack patterns |
| [certora-master-guide.md Section 9.5](certora-master-guide.md) | Attack synthesis workflow |

### Debugging (When Rules Fail)
| Document | Use When |
|----------|----------|
| [certora-ce-diagnosis-framework.md](certora-ce-diagnosis-framework.md) | Investigating counterexamples |
| [best-practices-from-certora.md](best-practices-from-certora.md) | Section 2: 5-step investigation |
| [quick-reference-v1.3.md](quick-reference-v1.3.md) | Quick troubleshooting |

### Performance & CLI
| Document | Use When |
|----------|----------|
| [advanced-cli-reference.md](advanced-cli-reference.md) | Timeout optimization, advanced flags, `--method` name-only (v8.8.0+) |

### Reference & Learning
| Document | Use When |
|----------|----------|
| [tutorial-extraction-summary.md](tutorial-extraction-summary.md) | Understanding tutorial integration |
| [version-history.md](version-history.md) | Checking what's new, migration guides |
| [quick-reference-v1.3.md](quick-reference-v1.3.md) | Need printable cheat sheet |

---

## Quick Access by Phase

### Phase 0: Contract Analysis
**Primary:** [certora-master-guide.md](certora-master-guide.md) Section 3  
**Reference:** [SPEC AUTHORING (CERTORA).md](spec-authoring-certora.md) Phase 0  
**Checklist:** Enumerate entry points, storage, external calls

### Phase -1: Execution Closure
**Primary:** [certora-master-guide.md](certora-master-guide.md) Section 4  
**Reference:** [SPEC AUTHORING (CERTORA).md](spec-authoring-certora.md) Phase -1  
**Checklist:** Complete interaction ownership table

### Phase 2: Property Discovery
**Primary:** [categorizing-properties.md](categorizing-properties.md)  
**Enhanced:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 1  
**Tips:** Avoid 4 fatal mistakes, use dual mindset, mine tests  
**Checklist:** All properties in plain English + prioritized

### Phase 2.5: Classification (Invariant vs Rule)
**Primary:** [certora-master-guide.md](certora-master-guide.md) Section 6  
**Reference:** [certora-workflow.md](certora-workflow.md) Step 4  
**Checklist:** Each property classified with reasoning

### Phase 3.5: Causal Validation
**Primary:** [certora-master-guide.md](certora-master-guide.md) Section 7  
**Reference:** [certora-workflow.md](certora-workflow.md) Step 5  
**Checklist:** Reachability (`satisfy`) PASSED + Validation spec PASSED  
**Key (v1.8):** Write `satisfy` rules BEFORE assert rules — proves functions are live (anti-vacuity)

### Phase 4-6: Modeling & Sanity
**Primary:** [certora-master-guide.md](certora-master-guide.md) Section 8  
**Patterns:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 3 & 4  
**Checklist:** Sanity gate all checked

### Phase 7: Write CVL
**Primary:** [certora-spec-framework.md](certora-spec-framework.md)  
**CVL Reference:** [cvl-language-deep-dive.md](cvl-language-deep-dive.md) ⭐ **NEW v1.5**  
**Worked Examples:** [verification-playbooks.md](verification-playbooks.md) ⭐ **NEW v1.5**  
**Guide:** [certora-master-guide.md](certora-master-guide.md) Section 9  
**Patterns:** [best-practices-from-certora.md](best-practices-from-certora.md) Sections 3-5  
**Checklist:** Pre-verification checklist in QUICK_REFERENCE

### Phase 8: Attack Synthesis (Offensive) ⭐ NEW v3.0
**Primary:** [impact-spec-template.md](impact-spec-template.md)  
**Attack Patterns:** [multi-step-attacks-template.md](multi-step-attacks-template.md)  
**Guide:** [certora-master-guide.md](certora-master-guide.md) Section 9.5  
**Checklist:**
- [ ] Import impact tracking ghosts
- [ ] Run `attacker_cannot_profit` rule
- [ ] Run `system_value_conserved` rule
- [ ] Run multi-step attack patterns
- [ ] Convert any CEs to Foundry PoCs

### Debugging: Counterexamples
**Primary:** [certora-ce-diagnosis-framework.md](certora-ce-diagnosis-framework.md)  
**Workflow:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 2  
**Tips:** 5-step investigation, break down complex expressions

---

## Find Information By Topic

### Property Discovery
- **Main Guide:** [categorizing-properties.md](categorizing-properties.md)
- **Techniques:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 1
- **Fatal Mistakes:** BEST_PRACTICES Section 1.1
- **Prioritization:** Categorizing_Properties Section 7

### Invariant Design
- **Patterns:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 3
- **Monotonicity:** BEST_PRACTICES Section 3.2
- **Conservation Laws:** BEST_PRACTICES Section 3.2
- **Preserved Blocks:** BEST_PRACTICES Section 3.3

### Harness Creation
- **Best Practices:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 4
- **Templates:** [certora-master-guide.md](certora-master-guide.md) Section 11.1
- **When to Use:** BEST_PRACTICES Section 4.1

### Loop Handling
- **Strategy:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 5
- **Flags:** BEST_PRACTICES Section 5.1
- **Configuration:** CERTORA_MASTER_GUIDE Section 9.2

### Ghost Variables
- **Design:** [certora-master-guide.md](certora-master-guide.md) Section 8.2
- **Patterns:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 3.2
- **Hooks:** CERTORA_SPEC_FRAMEWORK hooks section

### CVL Syntax
- **Complete Reference:** [cvl-language-deep-dive.md](cvl-language-deep-dive.md) ⭐ **NEW v1.5** (20 sections)
- **Templates & Patterns:** [certora-spec-framework.md](certora-spec-framework.md)
- **Quick Lookup:** [quick-reference-v1.3.md](quick-reference-v1.3.md) Section 12.2
- **Worked Examples:** [verification-playbooks.md](verification-playbooks.md) ⭐ **NEW v1.5**

### Counterexample Debugging
- **Framework:** [certora-ce-diagnosis-framework.md](certora-ce-diagnosis-framework.md)
- **5-Step Process:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 2
- **Call Trace Analysis:** BEST_PRACTICES Section 2.2

### Common Issues
- **Pitfalls:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 6
- **Troubleshooting:** [certora-master-guide.md](certora-master-guide.md) Section 12.4
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

**See:** [tutorial-extraction-summary.md](tutorial-extraction-summary.md) for complete mapping

### Framework Evolution
- **v1.0 → v1.5:** [version-history.md](version-history.md)
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
- [ ] **Phase 3.5** - Reachability (`satisfy`) PASSED (MASTER_GUIDE Section 7) ← v1.8
- [ ] **Phase 3.5** - Validation spec PASSED (MASTER_GUIDE Section 7)
- [ ] **Phase 4-6** - Modeling complete, sanity checks passed (MASTER_GUIDE Section 8)
- [ ] **Phase 7** - Real spec written (MASTER_GUIDE Section 9)
- [ ] **Run** - Prover executed (MASTER_GUIDE Section 10)
- [ ] **Debug** - Counterexamples investigated (CE_DIAGNOSIS_FRAMEWORK)
- [ ] **Done** - All properties verified ✅

---

## Templates & Examples

### Copy-Paste Templates
- **Project Setup:** [certora-quickstart-template.md](certora-quickstart-template.md) Section 1
- **Property Template:** [categorizing-properties.md](categorizing-properties.md) Property Documentation Template
- **Prioritization Template:** Categorizing_Properties Section 7.2
- **CVL Spec Structure:** [certora-master-guide.md](certora-master-guide.md) Section 9.1
- **Harness Template:** CERTORA_MASTER_GUIDE Section 11.1
- **Bug Report Template:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 2.3

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

**Full Reference:** [quick-reference-v1.3.md](quick-reference-v1.3.md) or MASTER_GUIDE Section 10.1

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
1. **Common Pitfalls:** [best-practices-from-certora.md](best-practices-from-certora.md) Section 6
2. **Troubleshooting:** [certora-master-guide.md](certora-master-guide.md) Section 12.4
3. **CE Debugging:** [certora-ce-diagnosis-framework.md](certora-ce-diagnosis-framework.md)

### Chat Assistant Prompts
Use MASTER_GUIDE Section 13 for ready-to-use prompts for each phase.

---

## Framework Files Summary

Total: **19 documents** (v1.5)

**Core (8):**
1. readme.md
2. certora-master-guide.md
3. certora-workflow.md
4. certora-spec-framework.md
5. certora-ce-diagnosis-framework.md
6. SPEC AUTHORING (CERTORA).md
7. categorizing-properties.md
8. certora-quickstart-template.md

**Enhanced (3 — added v1.3):**
9. best-practices-from-certora.md
10. quick-reference-v1.3.md
11. tutorial-extraction-summary.md

**v1.4 Additions (4):**
12. advanced-cli-reference.md ⭐ CLI & performance
13. POC_TEMPLATE_Foundry.md ⭐ Foundry PoC template
14. POC_TEMPLATE_HARDHAT.md ⭐ Hardhat PoC template
15. VULNERABILITY_REPORT_TEMPLATE.md ⭐ Report template

**v1.5 Additions (2):**
16. cvl-language-deep-dive.md ⭐ Complete CVL language reference
17. verification-playbooks.md ⭐ ERC-20/WETH/ERC-721 worked examples

**Meta (2):**
18. version-history.md
19. index.md (this file)

---

**Framework Version:** 1.9 (Red Team Hardening)  
**Status:** Production-Ready  
**Last Updated:** February 16, 2026

**Start Your Verification Journey:** [certora-master-guide.md](certora-master-guide.md)
