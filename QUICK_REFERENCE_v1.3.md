# Framework v1.3 Quick Reference Card

> **Print this or keep open while verifying**

---

## 📋 Phase Checklist with Best Practices

### Phase 2: Discover Properties

- [ ] Read interface & NatSpec comments
- [ ] Write properties in plain English
- [ ] Apply Dual Mindset (Should Always / Should Never)
- [ ] Mine existing tests for implicit properties
- [ ] **[NEW v1.3]** Assign priority (HIGH / MEDIUM / LOW)
- [ ] Categorize (Valid State / Transition / System / Threat)
- [ ] ⚠️ **DON'T** mimic implementation code
- [ ] ⚠️ **DON'T** copy requires from contract

**Reference:** BEST_PRACTICES Section 1

---

### Phase 3.5: Decide Invariants

- [ ] Identify always-true conditions
- [ ] Check if state tracking needed (ghost variables)
- [ ] Apply invariant design patterns:
  - [ ] Monotonicity (counters never decrease)
  - [ ] Per-entity bounds (each user's value valid)
  - [ ] Relationships (part < whole)
  - [ ] Conservation (total = sum of parts)
- [ ] Document invariant dependencies

**Reference:** BEST_PRACTICES Section 3

---

### Phase 7: Write CVL

- [ ] Start with HIGH priority properties
- [ ] Break down complex expressions
- [ ] Add descriptive error messages
- [ ] Use `@withrevert` for failure cases
- [ ] Include preserved blocks with dependencies
- [ ] Set realistic bounds in requires

**Reference:** BEST_PRACTICES Section 6.2

---

### CE Debugging: Investigate Violations

**5-Step Process:**
1. [ ] Run entire spec first (get overview)
2. [ ] Focus on one rule (`--rule rule_name`)
3. [ ] Analyze call trace:
   - [ ] Check storage before/after
   - [ ] Check arguments passed
   - [ ] Check return values
   - [ ] Look for HAVOC annotations
4. [ ] Identify deviation (spec bug vs real bug)
5. [ ] Fix and document

**Reference:** BEST_PRACTICES Section 2, CE_DIAGNOSIS_FRAMEWORK

---

## ⚠️ Common Pitfalls to Avoid

### The Four Fatal Mistakes

| Mistake | How to Avoid |
|---------|--------------|
| **Wrong Property** | Write in plain English first, validate logic |
| **Partial Coverage** | Use comprehensive categorization |
| **Duplicate Property** | Track property IDs, review overlaps |
| **Mimicking Implementation** | Write from spec, NOT from code |

### Spec Readability

```cvl
// ❌ BAD
assert balanceOf(a) + debt[a] <= totalSupply() + totalDebt();

// ✅ GOOD
uint256 userTotal = balanceOf(a) + debt[a];
uint256 systemTotal = totalSupply() + totalDebt();
assert userTotal <= systemTotal;
```

---

## 🎯 Priority Guidelines

| Priority | Criteria | Example |
|----------|----------|---------|
| **HIGH** | Loss of funds, DoS, privilege escalation | "Winner cannot claim" |
| **MEDIUM** | Accounting integrity, solvency | "Total supply tracking" |
| **LOW** | Single function behavior | "mint() increases balance" |

**Verification Order:**
1. Simple HIGH properties (quick wins)
2. Complex HIGH properties (critical)
3. MEDIUM properties (if time permits)
4. LOW properties (best effort)

---

## 🔧 Loop Handling

```json
{
    "loop_iter": "5",        // Realistic max + 1
    "optimistic_loop": true  // Assume correct completion
}
```

⚠️ Set `loop_iter` high enough to cover realistic scenarios!

---

## 🏗️ Harness Patterns

```solidity
contract MyContractHarness is MyContract {
    // Expose internal state
    function getInternalVar() external view returns (uint256) {
        return internalVar;
    }
    
    // Helper for verification
    function sumOfBalances(address[] memory users) 
        external view returns (uint256) 
    {
        uint256 sum = 0;
        for (uint i = 0; i < users.length; i++) {
            sum += balances[users[i]];
        }
        return sum;
    }
}
```

---

## 📊 Invariant Patterns Cheat Sheet

### Monotonicity (Value Never Decreases)

```cvl
ghost mathint totalGhost {
    init_state axiom totalGhost == 0;
}

hook Sstore var uint256 newVal (uint256 oldVal) {
    totalGhost = totalGhost + (newVal - oldVal);
}

invariant monotonicity() totalGhost >= 0
```

### Conservation (Total = Sum)

```cvl
ghost mapping(address => mathint) userAmounts;
ghost mathint totalAmount;

invariant conservation()
    totalAmount == sum_of_userAmounts
```

### Relationship (Part < Whole)

```cvl
invariant partLessThanWhole(address user)
    userBalance[user] <= totalSupply()
```

---

## ⚡ Performance & Advanced Flags

**Reference:** ADVANCED_CLI_REFERENCE.md for complete guide

### Timeout Mitigation

| Problem | Solution | Command |
|---------|----------|---------|
| Multiple slow rules | Split to separate jobs | `--split_rules "pattern"` |
| Complex assertions | Check separately | `--multi_assert_check` |
| High path count | Control flow splitting | `--prover_args '-depth N'` |
| Loops timing out | Adjust iterations | `--loop_iter N` |

### Advanced Debugging

| Need | Flag | Use Case |
|------|------|----------|
| Multiple CEs | `--multi_example` | See different failure paths |
| Multiple scenarios | `--independent_satisfy` | Check satisfy statements separately |
| Vacuity check | `--rule_sanity basic` | Ensure rules can fail |
| Coverage gaps | `--coverage_info advanced` | Find untested code |

### Quick Performance Commands

```bash
# Split heavy rules
certoraRun config.conf --split_rules "solvency_*"

# Multi-assert for timeouts
certoraRun config.conf --multi_assert_check

# Eager splitting for large code
certoraRun config.conf --prover_args '-smt_initialSplitDepth 5 -depth 15'

# Multiple counterexamples
certoraRun config.conf --rule failing_rule --multi_example
```

---

## 🚀 Before Running Prover

```
□ Properties in plain English first?
□ Properties don't mimic implementation?
□ Properties prioritized by impact?
□ Complex expressions broken down?
□ Preserved blocks include dependencies?
□ Loop_iter set appropriately?
□ Harnesses expose needed state?
□ Property IDs tracked (no duplicates)?
```

---

## 📚 Document Quick Links

| Need | Document | Section |
|------|----------|---------|
| Performance optimization | **ADVANCED_CLI_REFERENCE** | **Sections 1-2** |
| Loop/array handling | **ADVANCED_CLI_REFERENCE** | **Section 3** |
| Multi-version projects | **ADVANCED_CLI_REFERENCE** | **Section 4** |
| Property discovery | BEST_PRACTICES | Section 1 |
| CE investigation | BEST_PRACTICES | Section 2 |
| Invariant patterns | BEST_PRACTICES | Section 3 |
| Harness design | BEST_PRACTICES + ADVANCED_CLI | Sections 4, 6 |
| Loop handling | BEST_PRACTICES + ADVANCED_CLI | Sections 5, 3 |
| Common mistakes | BEST_PRACTICES | Section 6 |
| Prioritization | Categorizing_Properties | Section 7 |
| CVL syntax | CERTORA_SPEC_FRAMEWORK | All |
| Full workflow | CERTORA_MASTER_GUIDE | All |

---

## 🎓 Tutorial Mapping

| Your Phase | Relevant Tutorial |
|------------|-------------------|
| Phase 2 | Lesson 06 (Thinking Properties) |
| Phase 3.5 | Lessons 07-08 (Invariants) |
| Phase 5 | Lessons 03-05 (Ghosts/Hooks) |
| Phase 7 | Lesson 01 (Basic Rules) |
| Phase 11 | Lesson 15 (Harness) |
| CE Debug | Lesson 02 (Investigate) |
| Loops | Lesson 11 (Loops) |

---

## 💡 Key Quotes

> "Coming up with meaningful properties is the most challenging part of the work."
> — Certora Tutorial Lesson 06

> "Try breaking complex expressions to achieve code readability and a more simplified call trace."
> — Certora Tutorial Lesson 02

> "Without the ability to identify and express meaningful properties, all your technical knowledge with the tool is worthless."
> — Certora Tutorial Lesson 06

---

**Framework Version:** 1.3  
**Last Updated:** With Certora Tutorial best practices integration
