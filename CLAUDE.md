# Formal Verification Workspace

> **Instructions** — Read this before taking any action in this repo.

---

## What This Workspace Is

This is a Solidity smart-contract codebase (`contracts/`) under formal verification using the **Certora Prover** (CVL 2.x). The verification effort is guided by the **Certora FV Framework v3.2**, a plain-markdown methodology knowledge base located at `Certora-Fv-Framework/`.

The framework is **version 3.2 (Optimization Pressure + Temporal Depth + Design Hostility)**. It operates in a mandatory **Offensive ⇄ Defensive feedback loop**. Proof comes last, always last.

---

## Start Here — Framework Entry Point

Do not guess at framework conventions. Always begin by reading:

```
Certora-Fv-Framework/index.md        ← Master navigation (not README.md)
Certora-Fv-Framework/readme.md       ← Framework mission & philosophy
```

The `index.md` is the authoritative navigation hub. It maps every document to its phase and purpose.

---

## Framework Philosophy (Non-Negotiable)

```
UNDERSTAND → ENUMERATE → VALIDATE → ATTACK ⇄ DEFEND → PROVE
```

**Core principle:** Assume the design is hostile. Question who benefits. Search for maximum extractable value. Prove safety only AFTER failing to break it.

**Two modes:**

| Mode | Purpose | Primary Documents |
|------|---------|------------------|
| OFFENSIVE | Discover profitable attack paths | `impact-spec-template.md`, `multi-step-attacks-template.md`, `offensive-pipeline.md` |
| DEFENSIVE | Prove discovered attacks are fixed | `certora-spec-framework.md`, `cvl-language-deep-dive.md` |

---

## Phase Ordering & Hard Gates

Never skip or reorder phases. Each gate must pass before proceeding.

| Phase | Name | Primary Document | Hard Gate |
|-------|------|-----------------|-----------|
| 0 | Contract Analysis | `certora-master-guide.md` §3 | All entry points, storage, external calls enumerated |
| -1 | Execution Closure | `certora-master-guide.md` §4 | Full interaction ownership table complete |
| 2 | Property Discovery | `categorizing-properties.md` | Properties in plain English, prioritized |
| 2.5 | Invariant vs Rule Classification | `certora-master-guide.md` §6 | Each property classified with reasoning |
| 3.5 | Causal Validation | `certora-master-guide.md` §7 | `satisfy` rules PASS (reachability) AND validation spec PASS |
| 4–6 | Modeling & Sanity | `certora-master-guide.md` §8 | All sanity checks pass |
| §8.4 | Adversarial Design Interrogation | `certora-master-guide.md` §8.4 | **MANDATORY** — 5 questions answered before any spec is written |
| 7 ⇄ 8 | Offensive ⇄ Defensive Loop | `certora-master-guide.md` §9 & §9.5 | Loop converged, profit boundary established (§9.5.10) |
| Final | Defensive Proof | `certora-spec-framework.md` | Written and passing — **ALWAYS THE LAST STEP** |

### Phase 3.5 anti-vacuity rule
Write `satisfy` (reachability) rules **before** `assert` rules. This proves functions are live and prevents vacuous proofs.

### §9.5.10 Profit Escalation
Do not stop at existence of an exploit. Find the **maximum extractable value** via iterative threshold escalation (SAT→UNSAT boundary). An UNSAT result at the first threshold is not sufficient — the boundary must be established.

---

## Document Map

### Core Methodology
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/certora-master-guide.md` | Complete 9-phase step-by-step process (3334 lines — read sections as needed) |
| `Certora-Fv-Framework/spec-authoring-certora.md` | Deep methodology & the WHY behind every decision |
| `Certora-Fv-Framework/certora-workflow.md` | Visual workflow with Mermaid diagrams |
| `Certora-Fv-Framework/certora-quickstart-template.md` | **Copy this** to start a new verification project |

### Property Discovery (Phases 0–6)
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/categorizing-properties.md` | Property discovery, dual mindset, 4 fatal mistakes |
| `Certora-Fv-Framework/best-practices-from-certora.md` | Official Certora tutorial techniques |

### CVL Writing (Phase 7) — Source of Truth
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/cvl-language-deep-dive.md` | **Authoritative CVL 2.x reference** — type system, ghosts, hooks, invariants, builtin rules (20 sections). Use this, do not hallucinate CVL syntax. |
| `Certora-Fv-Framework/certora-spec-framework.md` | CVL patterns, templates, spec structure |
| `Certora-Fv-Framework/verification-playbooks.md` | Worked examples: ERC-20, WETH, ERC-721 — copy-paste patterns |
| `Certora-Fv-Framework/quick-reference-v1.3.md` | Quick syntax lookup cheat sheet |

### Offensive Verification (Phase 8)
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/impact-spec-template.md` | **Copy this** for anti-invariant & profit threshold rules |
| `Certora-Fv-Framework/multi-step-attacks-template.md` | Flash loan, sandwich, staged, cross-contract attack patterns |
| `Certora-Fv-Framework/offensive-pipeline.md` | `.conf` setup, CE triage, attack prioritization, severity matrix |

### Debugging
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/certora-ce-diagnosis-framework.md` | Counterexample debugging — the only CE debug workflow |
| `Certora-Fv-Framework/best-practices-from-certora.md` §2 | 5-step investigation, call trace analysis |
| `certora-call-trace-debugging-tips.md` | Repo-specific call trace tips |

### PoC Conversion & Reporting
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/poc-template-foundry.md` | Convert Certora CE → Foundry exploit PoC |
| `Certora-Fv-Framework/poc-template-hardhat.md` | Convert Certora CE → Hardhat exploit PoC |
| `Certora-Fv-Framework/vulnerability-report-template.md` | **Copy this** for writing bug reports |

### Performance & CLI
| Document | Purpose |
|----------|---------|
| `Certora-Fv-Framework/advanced-cli-reference.md` | Timeout optimization, `--method` name-only (v8.8.0+), advanced flags |

---

## Forbidden Actions

These are the most common mistakes AI agents make in this framework. Never do them.

- **Never write CVL before completing Phases 0 through 3.5.** Causal validation defines what is provable.
- **Never skip the §8.4 Adversarial Design Interrogation.** It is mandatory before any spec is written.
- **Never write a full defensive spec before an offensive spec exists.** They evolve together.
- **Never trust UNSAT on an offensive rule until assumptions are verified as non-artificial.** Overly strong assumptions produce false UNSAT.
- **Never skip causal validation.** Skipping `satisfy` rules leads to vacuous proofs that prove nothing.
- **Never assume external contracts are "standard"** without proving they match the expected interface.
- **Never treat the first UNSAT threshold as the profit boundary.** Run the full §9.5.10 escalation protocol.
- **Never create spec files from scratch** when a template exists. Copy `certora-quickstart-template.md` or `impact-spec-template.md`.
- **Never hallucinate CVL syntax.** Always look it up in `cvl-language-deep-dive.md`.

---

## Common Commands

```bash
# Clear prover cache (always run before a clean verification)
rm -rf .certora_internal

# Run a validation spec (causal validation — Phase 3.5, always first)
certoraRun certora/confs/validation_<Contract>.conf

# Run a real spec
certoraRun certora/confs/<Contract>.conf

# Run a single rule (fast iteration)
certoraRun certora/confs/<Contract>.conf --rule "ruleName"

# Run offensive spec
certoraRun certora/confs/offensive.conf

# Run Foundry tests (PoC validation)
forge test --match-contract <ExploitTest> -vvv
```

Full reference: `Certora-Fv-Framework/advanced-cli-reference.md` and `Certora-Fv-Framework/quick-reference-v1.3.md`.

---

## Where to Find Things

| I need to... | Go to |
|---|---|
| Start a new verification from scratch | Copy `certora-quickstart-template.md`, then follow `certora-master-guide.md` |
| Understand the phase I'm in | `certora-master-guide.md` table of contents |
| Write a `satisfy` reachability rule | `certora-master-guide.md` §7 + `cvl-language-deep-dive.md` |
| Write a ghost variable | `cvl-language-deep-dive.md` §5 (ghosts & hooks) |
| Write an invariant | `cvl-language-deep-dive.md` §4 + `certora-spec-framework.md` |
| Write an offensive profit rule | Copy `impact-spec-template.md` |
| Debug a counterexample | `certora-ce-diagnosis-framework.md` |
| Convert a CE to a Foundry PoC | `poc-template-foundry.md` |
| Optimize timeout / performance | `advanced-cli-reference.md` |
| Write a vulnerability report | Copy `vulnerability-report-template.md` |
| Understand what's new in v3.2 | `Certora-Fv-Framework/readme.md` §"What's New in v3.2" |
| Check quick CVL syntax | `quick-reference-v1.3.md` |

---

## Framework Version

**Certora FV Framework:** v3.2 (Optimization Pressure + Temporal Depth + Design Hostility)
**CVL:** 2.x
**Prover CLI:** certoraRun (minimum version supporting `--method` name-only: v8.8.0)
**Last updated:** March 4, 2026

---

<!-- LOCAL CUSTOMIZATIONS — everything below this line is preserved on update -->
