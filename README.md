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
