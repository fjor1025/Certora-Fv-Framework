# Certora-Fv-Framework

```
/home/brett/dual-governance/
├── SPEC AUTHORING (CERTORA).md    # Deep methodology (Phase 0-6 details)
├── CERTORA_SPEC_FRAMEWORK.md      # CVL 2.0 templates & patterns  || Write specs correctly the first time
├── CERTORA_CE_DIAGNOSIS_FRAMEWORK.md  # Counterexample debugging || Debug counterexamples systematically
└── CERTORA_WORKFLOW.md            # Step-by-step execution guide ← NEW
```
**How they connect:**

* **WORKFLOW.md** — Follow aschecklist
* **SPEC AUTHORING** — Deep dive when I need more detail on a phase
* **SPEC FRAMEWORK** — Reference when writing actual CVL in Phase 7
* **CE DIAGNOSIS** — Reference when prover returns counterexamples

**Complete Framework Structure**

```bash
your-project/
├── CERTORA_MASTER_GUIDE.md         ← START HERE (just created)
├── CERTORA_WORKFLOW.md             ← Phase details
├── CERTORA_SPEC_FRAMEWORK.md       ← CVL templates
├── CERTORA_CE_DIAGNOSIS_FRAMEWORK.md ← Debug counterexamples
├── SPEC AUTHORING (CERTORA).md     ← Deep methodology
├── Categorizing_Properties.md      ← Phase 2 categorization
└── CVLDocs/                        ← Reference documentation
```
**To Apply to Any New Contract**
```bash
# 1. Set contract name
CONTRACT_NAME="MyNewContract"

# 2. Create structure (from Section 2.1)
mkdir -p spec_authoring certora/{specs,confs,harnesses,helpers}

# 3. Follow the guide phase by phase
```
## Section 13: Quick Start Chat Prompt (NEW)

**Copy-paste prompts for each phase:**

* **13.1** Brand new verification project
* **13.2** Continuing Phase 0 / Phase -1
* **13.3** Phase 2 (Property Discovery)
* **13.4** Phase 3.5 (Causal Validation)
* **13.5** Phase 7 (Writing CVL)
* **13.6** Debugging counterexamples
* **13.7** Essential information checklist

---

# 13. QUICK START CHAT PROMPT

> **Use this section when starting a new verification with an AI assistant.**  
> **Copy the appropriate prompt below and paste it into your chat.**

## 13.1 For a Brand New Verification Project

```markdown
I am starting a formal verification project using Certora for the following contract:

**Project Location:** [/path/to/project]
**Primary Target Contract:** [ContractName.sol at path/to/contract]
**Contract Dependencies:** [List the files that the target imports]

Please help me follow the CERTORA_MASTER_GUIDE.md workflow:

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

The framework documents are already in my project root.
```

## 13.2 For Continuing Phase 0 / Phase -1

```markdown
Continue the Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** [0 / -1]

Please analyze the contract and help me fill in the spec_authoring document:
- Phase 0: Entry points, storage variables, asset flows
- Phase -1: External contracts, interaction ownership table, modeling decisions

Reference: CERTORA_MASTER_GUIDE.md sections 3 and 4.
```

## 13.3 For Phase 2 (Property Discovery)

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** 2 (Property Discovery)

Based on the Phase 0/-1 analysis, help me discover security properties using Categorizing_Properties.md:
- Valid States (range constraints)
- State Transitions (function effects)
- System-Level (aggregates, sums)
- Access Control (who can do what)

Output should go into: spec_authoring/{target}_candidate_properties.md
```

## 13.4 For Phase 3.5 (Causal Validation)

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** 3.5 (Causal Validation)

Create the validation spec and conf to verify mutation paths are complete:
1. Create certora/specs/validation_{target}.spec
2. Create certora/confs/validation_{target}.conf
3. Include validation rules for each INVARIANT variable
4. Include ghost synchronization tests if ghosts are needed

Reference: CERTORA_MASTER_GUIDE.md section 7.
```

## 13.5 For Phase 7 (Writing CVL)

```markdown
Continue Certora verification for [ContractName]:

**Target:** [path/to/ContractName.sol]
**Current Phase:** 7 (Write CVL)

Prerequisites completed:
- [x] Validation spec PASSED
- [x] All properties classified (INVARIANT vs RULE)
- [x] Modeling decisions documented

Please create:
1. certora/specs/{ContractName}.spec with all invariants and rules
2. certora/confs/{ContractName}.conf 

Reference: CERTORA_MASTER_GUIDE.md section 9 and CERTORA_SPEC_FRAMEWORK.md.
```

## 13.6 For Debugging Counterexamples

```markdown
I have a failing rule in my Certora verification:

**Target:** [ContractName]
**Failing Rule:** [rule name]
**Error/CE Summary:** [paste the counterexample or error]

Please help me diagnose using CERTORA_CE_DIAGNOSIS_FRAMEWORK.md:
1. Is this a REAL bug or SPURIOUS result?
2. If spurious, what modeling is missing?
3. If real, what's the attack vector?
```

---

## 13.7 Essential Information to Provide

When starting any verification conversation, always include:

| Required Info | Example |
|---------------|---------|
| **Project path** | `/home/user/my-protocol` |
| **Target contract** | `contracts/core/Vault.sol` |
| **Contract name** | `Vault` (as declared in Solidity) |
| **Dependencies** | `imports Token.sol, Oracle.sol, Utils.sol` |
| **Current phase** | Phase 0 / -1 / 2 / 2.5 / 3.5 / 7 |

**Optional but helpful:**
- Known external integrations (ERC20, Chainlink, Uniswap, etc.)
- Special patterns (proxy, upgradeable, diamond)
- Existing tests or known issues

---
