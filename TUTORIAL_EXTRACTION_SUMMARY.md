# Tutorial Extraction Summary

> **What:** Key techniques extracted from official Certora Tutorials  
> **Why:** Enhance framework with battle-tested best practices  
> **Result:** Framework v1.5 with CVL deep dive, verification playbooks, prioritization, investigation workflow, and comprehensive best practices

---

## Extraction Sources

### Tutorials Analyzed

| Lesson | Topic | Key Extractions |
|--------|-------|-----------------|
| **Lesson 02** | Investigate Violations | 5-step CE investigation workflow |
| **Lesson 06** | Thinking Properties | Property discovery, 4 fatal mistakes |
| **Lesson 07** | Inductive Reasoning | Preconditions, postconditions, invariants |
| **Lesson 08** | Working With Invariants | Preserved block patterns |
| **Lesson 11** | Loops | Loop unrolling, --optimistic_loop flag |
| **Lesson 15** | Harness | Harness design patterns with examples |
| **Auction Demo** | propertiesList.md | Property prioritization (HIGH/MEDIUM/LOW) |

---

## Integration Map

### Where Techniques Were Added

```
Framework Document                    Section Added                Tutorial Source
─────────────────────────────────────────────────────────────────────────────────
BEST_PRACTICES_FROM_CERTORA.md       [NEW FILE]                   All tutorials
├── Section 1                         Property Discovery           Lesson 06
├── Section 2                         CE Investigation             Lesson 02
├── Section 3                         Invariant Patterns           Lessons 07, 08, 15
├── Section 4                         Harness Best Practices       Lesson 15
├── Section 5                         Loop Handling                Lesson 11
└── Section 6                         Common Pitfalls              Lesson 06

Categorizing_Properties.md           Section 7                     Auction Demo
└── Property Prioritization          NEW in v1.3                   propertiesList.md

CERTORA_CE_DIAGNOSIS_FRAMEWORK.md    Tutorial-Based Workflow       Lesson 02
└── 5-Step Investigation             NEW in v2.1                   Investigate Violations

README.md                            Version & File List           N/A
└── Updated to v1.3                  Framework overview
```

---

## Key Insights Extracted

### 1. Property Discovery is the Hardest Part

> **Certora Tutorial Lesson 06:**
> "Coming up with meaningful properties is the most challenging part of the work. Without the ability to identify and express meaningful properties, all your technical knowledge with the tool is worthless."

**Framework Impact:** Added emphasis on property discovery in Phase 2, documented common mistakes.

### 2. The Four Fatal Mistakes

| Mistake | Impact | Prevention |
|---------|--------|------------|
| Wrong Property | Unhelpful counterexamples | Write in plain English first |
| Partial Coverage | Bugs slip through | Use comprehensive categorization |
| Duplicate Property | Wasted time | Track property IDs |
| Mimicking Implementation | Verifies code replicates bug | Write from spec, not code |

**Framework Impact:** Added anti-patterns section to BEST_PRACTICES.

### 3. Prioritization Prevents Waste

**Auction Demo Insight:** Not all properties have equal impact. Prioritize by:
- HIGH: Loss of funds, DoS, privilege escalation
- MEDIUM: Accounting integrity, solvency
- LOW: Single function correctness, redundant checks

**Framework Impact:** Added Section 7 to Categorizing_Properties.md with prioritization templates.

### 4. CE Investigation is Systematic

**Tutorial Lesson 02 Process:**
1. Run entire spec (get overview)
2. Focus on one rule (--rule flag)
3. Analyze call trace (storage, args, returns)
4. Identify deviation (spec vs implementation)
5. Fix and verify (document decision)

**Framework Impact:** Added systematic workflow to CE_DIAGNOSIS_FRAMEWORK.

### 5. Invariant Design Patterns Exist

**From Lesson 15 (Borda Election):**
- Monotonicity patterns (counters never decrease)
- Per-entity monotonicity (each user's value non-negative)
- Relationship invariants (part < whole)
- Conservation laws (total = sum of parts)

**Framework Impact:** Added Section 3.2 with ghost-based invariant patterns.

---

## Practical Examples Added

### Example 1: Monotonicity Invariant

```cvl
/// @title Points are non-decreasing
ghost mathint totalPointsGhost {
    init_state axiom totalPointsGhost == 0;
}

hook Sstore points uint256 newPoints (uint256 oldPoints) {
    totalPointsGhost = totalPointsGhost + (newPoints - oldPoints);
}

invariant pointsMonotonicity()
    totalPointsGhost >= 0
```

### Example 2: Priority Documentation

```markdown
### HIGH - VS2: Winner Can Always Claim Prize

**Impact:** Loss of funds
**Attack Vector:** Funds locked if winner cannot withdraw
**Reasoning:** Direct user fund loss violates core protocol guarantee
**Verification Cost:** Medium
```

### Example 3: Breaking Down Complex Expressions

```cvl
// ❌ Complex (hard to debug)
assert balanceOf(user) + debt[user] <= totalSupply() + totalDebt();

// ✅ Broken down (easier call trace)
uint256 userTotal = balanceOf(user) + debt[user];
uint256 systemTotal = totalSupply() + totalDebt();
assert userTotal <= systemTotal, "User total exceeds system total";
```

---

## Framework Version History

### v1.5 (Current) - RareSkills Integration
- CVL Language Deep Dive (20-section CVL reference)
- Verification Playbooks (ERC-20/WETH/ERC-721 worked examples)
- Vacuous truth defense, requireInvariant lifecycle, self-transfer handling
- Ghost havocing diagnosis, Liveness/Effect/No-Side-Effect pattern
- Definition blocks template, updated chat prompts

### v1.4 - Performance Optimization + CLI
- Advanced CLI Reference (timeout mitigation, performance flags)
- PoC templates (Foundry + Hardhat)
- Vulnerability Report template

### v1.3 - Priority-Enhanced + Tutorial Best Practices
- Property prioritization system (HIGH / MEDIUM / LOW)
- 5-step CE investigation workflow
- Comprehensive best practices document
- Invariant design patterns
- Harness best practices
- Loop handling strategies

### v1.2 - Dual Mindset + Test Mining
- "Should Always" / "Should Never" enumeration
- Test-to-property translation patterns
- Coverage blind spot identification

### v1.1 - Transition + Chat Prompts
- Validation-to-Real-Spec transition guide
- Per-phase chat prompts

### v1.0 - Initial Framework
- 9-phase methodology
- 6 core documents
- Complete CVL syntax reference

---

## Usage Guidelines

### When to Reference BEST_PRACTICES_FROM_CERTORA.md

| Phase | Use Case | Section |
|-------|----------|---------|
| **Phase 2** | Discovering properties | Section 1: Property Discovery |
| **Phase 2** | Avoiding common mistakes | Section 6: Common Pitfalls |
| **Phase 2** | Prioritizing properties | Reference Categorizing_Properties.md Section 7 |
| **Phase 3.5** | Designing invariants | Section 3: Invariant Patterns |
| **Phase 7** | Writing rules | Section 6.2: Readability |
| **Phase 11** | Creating harnesses | Section 4: Harness Best Practices |
| **CE Debugging** | Investigating violations | Section 2: CE Investigation |
| **Loop Issues** | Handling loops | Section 5: Loop Handling |

### Quick Checklist Integration

The BEST_PRACTICES document ends with a pre-verification checklist:

```
□ Properties written in plain English first?
□ Properties don't mimic implementation?
□ Properties prioritized by impact?
□ Complex expressions broken down?
□ Preserved blocks include dependencies?
□ Loop_iter set appropriately?
□ Harnesses expose needed internal state?
□ Property IDs tracked to avoid duplication?
```

Use this before running `certoraRun`.

---

## Next Steps for Framework Users

### For Your 3 Passing Validation Specs

You have 3 specs that passed validation:
1. Tokenomics
2. LiquidityManagerCore  
3. DefaultTargetDispenserL2

**Recommended Action:**
1. Apply Section 7 (Prioritization) to your candidate_properties.md files
2. Mark each property as HIGH / MEDIUM / LOW
3. Use prioritization to decide verification order in Phase 7
4. Reference BEST_PRACTICES Section 3 when designing invariants

### For New Contract Verifications

1. **Phase 2:** Use Categorizing_Properties.md + BEST_PRACTICES Section 1
2. **Phase 3.5:** Reference invariant patterns from Section 3.2
3. **Phase 7:** Follow readability guidelines from Section 6.2
4. **CE Debugging:** Use 5-step workflow from Section 2
5. **Throughout:** Check Section 6 (Common Pitfalls) to avoid mistakes

---

## Tutorial-to-Framework Mapping Reference

```
Certora Tutorial Lessons          Your Framework Phases
────────────────────────────────────────────────────────
Lesson 01: Basic Rules      →     Phase 7: Write CVL
Lesson 02: Investigate      →     CE Debugging
Lesson 03-05: Ghosts/Hooks  →     Phase 5: Ghost Design
Lesson 06: Properties       →     Phase 2: Discovery
Lesson 07-08: Invariants    →     Phase 3.5: Decide Invariants
Lesson 09-10: Summaries     →     Phase 6: Methods Block
Lesson 11: Loops            →     Phase 12: Prover Config
Lesson 12-14: Advanced      →     Phase 8+: Complex Specs
Lesson 15: Harness          →     Phase 11: Harness Design

Auction Demo                →     Phase 2: Prioritization
```

---

## Conclusion

The framework now incorporates **battle-tested techniques** from Certora's official tutorials, including:
- ✅ Systematic property discovery methodology
- ✅ Priority-based verification strategy
- ✅ Structured CE investigation workflow  
- ✅ Proven invariant design patterns
- ✅ Harness best practices with examples
- ✅ Common pitfalls and anti-patterns

**Result:** A more robust, production-ready framework that combines:
- Your original 9-phase methodology
- Dual mindset approach (v1.2)
- Test mining capabilities (v1.2)
- Property prioritization (v1.3)
- Official Certora best practices (v1.3)
- Advanced CLI & performance optimization (v1.4)
- RareSkills Certora Book integration (v1.5)

Use `BEST_PRACTICES_FROM_CERTORA.md` as your **technique reference**, `CVL_LANGUAGE_DEEP_DIVE.md` as your **CVL language reference**, and `VERIFICATION_PLAYBOOKS.md` for **production-ready worked examples**.
