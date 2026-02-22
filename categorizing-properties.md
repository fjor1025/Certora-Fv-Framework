# CATEGORIZING CANDIDATE SECURITY PROPERTIES

*(Pre-Spec, Syntax-Free Phase)*

> **Version:** 3.0 (Offensive Verification + Impact-First Framing)

## Purpose

This phase enumerates **candidate security properties** that may later become invariants or rules.
At this stage, we **do not** decide:

* whether the property is an invariant or a rule
* how it is encoded in CVL
* what assumptions or constraints are required

We are only answering:

> **"If this property were violated, would it cause real harm?"**

**NEW v3.0 — Impact-First Framing:**

> Don't start with "what should be true."  
> Start with "what would an attacker want?"  
> Then forbid that outcome.

---

## 0. ECONOMIC IMPACT CATEGORIES (NEW v3.0)

> **Philosophy:** Hunt for impact, not mistakes. The largest DeFi hacks had no 
> reentrancy, no arithmetic bugs, no access control issues — just bad economic assumptions.

Before categorizing properties by *type*, first categorize by *economic impact*.

### Impact Taxonomy

| Impact Category | Definition | Example | Severity |
|-----------------|------------|---------|----------|
| **Value Extraction** | Attacker gains tokens/ETH from protocol | Flash loan + oracle manipulation | CRITICAL |
| **Insolvency** | Protocol owes more than it holds | Undercollateralized after liquidation | CRITICAL |
| **Share Dilution** | Attacker inflates their claim vs others | Mint shares without backing | HIGH |
| **Debt Socialization** | Losses pushed onto other users | Bad debt spread to pool | HIGH |
| **Liquidity Freeze** | Users blocked from withdrawing | Permanent fund lock | HIGH |
| **Governance Capture** | Attacker controls protocol decisions | Flash loan voting | MEDIUM |
| **Front-Running** | Value extracted via tx ordering | Sandwich attacks | MEDIUM |
| **Griefing** | Attacker causes cost without profit | DoS without extraction | LOW |

### Impact Property Template (NEW v3.0)

```markdown
### [ID]. [Short Name]

**Attacker Goal:** [What the attacker wants to achieve]
**Impact Category:** [Value Extraction / Insolvency / Share Dilution / etc.]
**Estimated Impact:** [$X loss potential / Percentage of TVL]
**Irreversible?:** [Yes — funds gone / No — admin can fix]

**Attack Scenario:**
1. [Step 1]
2. [Step 2]  
3. [Profit extracted]

**Plain English Property:**
> "[What must never happen to prevent this attack]"

**Variables Involved:** [List storage variables and their owners]
**Multi-Step Required?:** [Yes — needs multi_step_attacks template / No]
```

### Attacker Objective Checklist (NEW v3.0)

Ask these questions for EVERY function:

| Question | If Yes → Property Needed |
|----------|--------------------------|
| Can caller profit from this call? | `attacker_cannot_profit` |
| Can caller extract more than they deposited? | Value extraction guard |
| Can caller inflate their share/claim? | Share dilution guard |
| Can caller push losses onto others? | Debt socialization guard |
| Can caller block others from withdrawing? | Liquidity freeze guard |
| Can caller front-run other users? | MEV resistance property |
| Can caller manipulate price/oracle? | Oracle manipulation guard |
| Can caller bypass time locks? | Timelock enforcement |

### Example: Impact-First Property Discovery

**Protocol:** Lending Pool  
**Function:** `withdraw(uint256 amount)`

**Impact Analysis:**

| Attacker Goal | Possible? | Property |
|---------------|-----------|----------|
| Withdraw more than deposited | Check | `user can never withdraw more than their proportional share` |
| Withdraw during insolvency (taking others' funds) | Check | `withdrawal must fail if pool would become insolvent` |
| Front-run liquidations to extract value | Check | `liquidation profit bounded by protocol parameters` |
| Grief by causing others' withdrawals to fail | Check | `withdrawal cannot be blocked by malicious actor` |

---
## Integration with Workflow

| Phase | Document | What Happens |
|-------|----------|--------------|
| Phase 0 / -1 | `{contract}_spec_authoring.md` | Execution reality mapped |
| **Phase 2** | **This document** | **Properties discovered & categorized** |
| Phase 2.5 | `{contract}_spec_authoring.md` | INVARIANT vs RULE decision |
| Phase 3.5 | `{contract}_causal_validation.md` | Mutation paths validated |

> ⚠️ **Prerequisite:** Phase 0 and Phase -1 MUST be complete before using this document.
> You need the ownership table to know which truths each property depends on.

---

## Property Documentation Template

For EACH candidate property, document:

```markdown
### [ID]. [Short Name]
**Plain English:** [One sentence describing the law]
**Impact if Violated:** [Theft / Insolvency / Privilege Escalation / Griefing / Protocol Corruption]
**Category:** [Valid State / State Transition / System-Level / Threat-Driven]
**Variables Involved:** [List storage variables and their owners]
**External Truths Needed:** [List any external contract state this depends on]
**Aggregate/History Required?:** [Yes/No - flags if ghost may be needed later]
```

> 📌 The "External Truths Needed" field connects back to your Phase -1 ownership table.

---
## 1. VALID STATES — “Laws of Existence”

**Concept:**
These describe states that should **never exist** at any point in execution.

If the system can ever *be* in such a state, the protocol is already broken.

**Candidate questions to ask:**

* Can the contract be insolvent?
* Can two mutually exclusive flags be true at once?
* Can a user’s position exist in a logically impossible configuration?
* Can an enum variable hold an invalid value?
* Can a relationship between two variables be violated?

**Examples (Conceptual, Not Formal):**

* "The protocol must never owe more assets than it holds."
* "A loan must never be undercollateralized outside liquidation."
* "An account must not be both frozen and active simultaneously."
* "State enum must always be 0, 1, or 2."

**Likely Outcome in Phase 2.5:** Often becomes INVARIANT

> These properties describe **what the world is allowed to look like**, not how it changes.

---

## 2. STATE TRANSITIONS — “Laws of Motion”

**Concept:**
These describe **how state is allowed to change** between two moments in time.

They do **not** assert what is always true — they assert **what must not happen during a transition**.

**Candidate questions to ask:**

* What variables should only increase or only decrease?
* What state changes must be tied to specific actions?
* What must remain unchanged if the caller lacks authority?
* What transitions between states are forbidden?
* What is the expected effect of calling function X?

**Examples (Conceptual):**

* "A user's debt can only increase if they explicitly borrow."
* "Configuration values must not change when called by a non-admin."
* "A nonce must never decrease or repeat."
* "State can only go 0→1→2, never backward."
* "Calling deposit() must increase the user's balance."

**Likely Outcome in Phase 2.5:** Often becomes RULE

> These properties describe **movement**, not existence.

---

## 3. SYSTEM-LEVEL PROPERTIES — “Global Accounting Truths”

**Concept:**
Some security properties depend on **aggregate or historical truth** that the contract does not explicitly store.

These properties are about **system-wide conservation**, not individual variables.

**Candidate questions to ask:**

* Is there value conservation across the system?
* Does supply only change through explicit mechanisms?
* Does reward distribution depend on real input (time, deposits)?
* Does the sum of all user balances equal the total?
* Are there cross-user relationships that must hold?

**Examples (Conceptual):**

* "Total issued claims must correspond to real assets."
* "Rewards must not appear without passage of time or contribution."
* "Global balances must reconcile with external token balances."
* "Sum of all individual locked shares must equal total locked shares."

**🚨 Ghost Flag:** Properties in this category often require **ghosts** for aggregation.
Mark "Aggregate/History Required?: Yes" in your property template.

**Likely Outcome in Phase 2.5:** Often becomes INVARIANT (but requires ghost infrastructure)

> These properties often require *auxiliary tracking*, but that decision comes later.

---

## 4. THREAT-DRIVEN PROPERTIES — “Anti-Exploit Laws”

**Concept:**
Instead of starting from code structure, start from **attacker impact**.

Ask:

> *“If I were malicious, what outcome would I want?”*

Then forbid that outcome.

---

### A. Theft / Insolvency

**Candidate questions:**

* Can funds leave without authorization?
* Can accounting be inflated temporarily?
* Can value be created from nothing?

**Examples:**

* “Funds must not leave unless the protocol explicitly allows it.”
* “User balances must not increase without corresponding backing.”

---

### B. Privilege Escalation / Governance Abuse

**Candidate questions:**

* Can voting power be faked?
* Can time delays be bypassed?
* Can privileged actions be triggered early?

**Examples:**

* “Execution must respect governance delays.”
* “Voting power must reflect real ownership, not flash liquidity.”

---

### C. Griefing / Freezing

**Candidate questions:**

* Can a malicious user permanently block progress?
* Can liquidation or withdrawal be made impossible?
* Can gas usage be weaponized?

**Examples:**

* “Unhealthy positions must always be liquidatable.”
* “No user input should permanently lock funds.”

---

## Important Exclusion Rule (Threat Model)

If a candidate property **only fails because a trusted role behaves maliciously**, it must be explicitly marked:

> **OUT OF SCOPE — Trusted Role Behavior**

Do **not** discard it silently.

**Examples of trusted role assumptions:**
* Admin doesn't set malicious parameters
* Governance doesn't pass malicious proposals
* Oracle provides accurate prices
* Bridge relays correct messages

> 📌 **Document WHY it's out of scope.** This becomes part of your threat model documentation.

---

## External Dependency Warning

🚨 **CRITICAL:** Properties that depend on external contract state need special handling.

If your property references:
* ERC20 balances held by another contract
* Oracle prices
* AMM reserves
* Lending protocol state
* Any state owned by a contract you don't control

Then mark it clearly:

> **External Truth Dependency: [Contract] owns [variable]**

This connects to your **Phase -1 Interaction Ownership Table**.

**Why this matters:**
* External state can change unexpectedly (via other callers)
* HAVOC can make these properties appear violated
* May need DISPATCHER or explicit modeling
* Often becomes a RULE rather than INVARIANT in Phase 2.5

---

## Output of This Phase

The output is a **plain-English list of candidate laws**, each with:

* A unique ID (e.g., A1, B2, C1)
* A short name
* A one-sentence description
* The category (Valid State / State Transition / System-Level / Threat-Driven)
* The impact if violated
* Variables involved and their owners
* External truths needed (if any)
* Whether aggregation/history tracking is needed

**No CVL. No syntax. No constraints.**

---

## Example Output Format

```markdown
### A1. ETH Solvency
**Plain English:** The ETH balance of the contract must always be >= total claimable ETH owed to users.
**Impact if Violated:** Insolvency - users cannot withdraw
**Category:** Valid State
**Variables Involved:** 
  - `address(this).balance` (owned by EVM)
  - `_accounting.totalClaimableETH` (owned by this contract)
**External Truths Needed:** None (ETH balance is native)
**Aggregate/History Required?:** No

### C1. Total Equals Sum
**Plain English:** Total locked shares must equal sum of all individual user locked shares.
**Impact if Violated:** Accounting corruption
**Category:** System-Level
**Variables Involved:**
  - `_accounting.stETHTotals.lockedShares` (owned by this contract)
  - `_accounting.assets[user].stETHLockedShares` (owned by this contract, per-user)
**External Truths Needed:** None
**Aggregate/History Required?:** Yes (sum over mapping requires ghost)
```

---

## Mental Check (Before Moving On)

You are done with this phase **only if**:

* Every item can be explained to a non-CVL auditor
* Nothing mentions `invariant`, `rule`, `require`, `assert`, or `ghost`
* Each property clearly maps to a *real exploit if broken*
* Each property has a unique ID for tracking
* External dependencies are explicitly noted
* Aggregate requirements are flagged

---

## Transition to Phase 2.5

After completing this phase, you will take each property to the **INVARIANT vs RULE Decision Gate** in Phase 2.5.

The decision will use:
1. Your property description
2. Your ownership analysis (from Phase -1)
3. Your external truth flags

**Do NOT pre-decide invariant vs rule here.** That's Phase 2.5's job.

Properties marked "Aggregate/History Required?: Yes" will need ghost design in Phase 5.

---

### One-Line Litmus Test

> **If it already looks like CVL, you skipped a step.**

> 🚫 **Anti-Pattern:** Any statement that can be translated directly into CVL without additional modeling decisions is too concrete for this phase.

This guards the boundary.

---

## Category Mapping Reference

How categories here relate to categories in `SPEC AUTHORING (CERTORA).md`:

| This Document | SPEC AUTHORING Phase 2 | Notes |
|---------------|------------------------|-------|
| Valid States | A. State-Inductive Safety | Same concept |
| State Transitions | B. State Transition Properties | Same concept |
| System-Level | C. Global/Cross-User Properties | Same concept |
| Threat-Driven | D. Attack Surface Inversion | Same concept |

> 📌 The categories are semantic (what kind of harm), NOT encoding decisions (invariant vs rule).

---

## Common Mistakes to Avoid

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| "invariant: balance >= 0" | Contains CVL keyword | "Balance must never be negative" |
| Skipping external truth analysis | HAVOC will break your spec | Fill in "External Truths Needed" |
| Not flagging aggregates | You'll miss ghost requirements | Mark "Aggregate Required?: Yes" |
| Mixing two truth owners | Creates causally unclosed property | Split into separate properties |
| Assuming trusted roles | Hidden threat model assumption | Mark "OUT OF SCOPE" explicitly |
| No unique IDs | Can't track through phases | Use A1, A2, B1, C1 format |
| Not documenting revert conditions | Leaves failure behavior undefined ← NEW v1.6 | Add "MUST REVERT WHEN" to each function property |
| Only writing "SHOULD ALWAYS" | Misses attacker perspective and failure paths ← NEW v1.6 | Use triple: SHOULD ALWAYS + SHOULD NEVER + MUST REVERT WHEN |

---

## Quick Checklist

Before leaving this phase:

- [ ] Every property has a unique ID
- [ ] Every property has one-sentence plain English description
- [ ] Every property has impact assessment
- [ ] Every property is categorized (Valid State / Transition / System-Level / Threat)
- [ ] Every property lists variables involved with owners
- [ ] External dependencies are flagged
- [ ] Aggregate requirements are flagged
- [ ] Trusted role assumptions are marked OUT OF SCOPE
- [ ] No CVL syntax anywhere
- [ ] No invariant/rule decision made yet

---

## 5. RIGHT / WRONG BEHAVIOR DUAL CHECKLIST

> **"Formal verification proves execution reality, not intent."**
> 
> To capture both correctness and attack surface, enumerate properties using BOTH mindsets:

### The Dual Mindset Approach

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   VALIDATION MINDSET              ATTACKER MINDSET                      │
│   (What SHOULD happen)            (What MUST NEVER happen)              │
│                                                                          │
│   "This should always..."         "This should never..."                │
│   "When X, then Y"                "Even if X, never Y"                  │
│   "Only A can do B"               "No one except A can do B"            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Right Behavior Checklist (SHOULD ALWAYS)

For each function/state, ask:

| Question | Example Property |
|----------|------------------|
| What is the expected postcondition? | "After deposit(), balance increases by amount" |
| What relationships must hold? | "Total supply == sum of balances" |
| What conservation laws apply? | "ETH in == ETH out over time" |
| What ordering must be respected? | "Unlock only after lock period" |
| What authorizations are required? | "Only owner can pause" |

### Wrong Behavior Checklist (SHOULD NEVER)

For each function/state, ask:

| Question | Example Property |
|----------|------------------|
| What would an attacker want? | "Withdraw more than deposited" |
| What creates value from nothing? | "Mint without backing" |
| What bypasses time locks? | "Execute before delay expires" |
| What escalates privileges? | "Non-admin changes config" |
| What causes permanent lock? | "Funds stuck forever" |
| What breaks accounting? | "Balance increases without transfer" |
| What allows reentrancy? | "State inconsistent mid-call" |
| What front-runs users? | "Sandwich attack on swap" |

### Revert Behavior Checklist (MUST REVERT WHEN) ← NEW v1.6

> **Why this matters:** By default, the Certora Prover ignores revert paths.
> If you don't explicitly model revert conditions with `@withrevert`,
> the behavior is undefined in your spec. Proving WHY something reverts
> is as important as proving WHY it succeeds.

For each state-changing function, ask:

| Question | Example Property |
|----------|------------------|
| When should this function refuse to execute? | "withdraw() must revert when balance < amount" |
| What prevents unauthorized access? | "pause() must revert when caller != owner" |
| What overflow/underflow conditions cause failure? | "mint() must revert when totalSupply + amount > max_uint256" |
| What precondition violations cause failure? | "transfer() must revert when sender == address(0)" |
| What state conditions prevent execution? | "claimReward() must revert when system is paused" |
| What timing conditions prevent execution? | "execute() must revert when block.timestamp < unlockTime" |

> **Connection to CVL:**
> Each "MUST REVERT WHEN" property becomes a `@withrevert` + `lastReverted` assertion in Phase 7.
> The strongest form uses biconditional: `assert lastReverted <=> (condition1 || condition2 || ...)`
> This proves the function reverts **if and only if** the listed conditions hold.

### Template: Dual Property Documentation

```markdown
### [ID]. [Short Name]

**RIGHT (Should Always):**
> "When [trigger], [expected outcome] should always occur."

**WRONG (Should Never):**
> "Even if [adversarial condition], [bad outcome] must never occur."

**REVERT (Must Revert When):** ← NEW v1.6
> "Function [name] must revert when [failure condition]."

**Variables:** [list]
**Attack Vector if Wrong:** [describe exploit]
```

### Example: Dual Property

```markdown
### A3. Withdrawal Integrity

**RIGHT (Should Always):**
> "When a user withdraws, their balance decreases by exactly the withdrawn amount."

**WRONG (Should Never):**
> "Even with multiple rapid withdrawals, a user must never withdraw more than their balance."

**REVERT (Must Revert When):** ← NEW v1.6
> "withdraw(amount) must revert when: balance[user] < amount, OR user == address(0), OR system is paused."

**Variables:** userBalance[user], totalSupply
**Attack Vector if Wrong:** Drain contract via integer underflow or reentrancy
```

---

## 6. TEST MINING — Extracting Properties from Existing Tests

> **Goal:** Mine existing test suites for implicit invariants, threat assumptions, and developer blind spots.

### Why Mine Tests?

Tests reveal:
1. **Core Invariants** - What developers thought was important
2. **Threat Assumptions** - What attacks they considered
3. **Blind Spots** - What they DIDN'T test

### Test Mining Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   STEP 1: Identify Test Files                                           │
│   ─────────────────────────────                                         │
│   Look for: test/, tests/, spec/, *.t.sol, *.test.js                   │
│                                                                          │
│   STEP 2: Categorize Test Types                                         │
│   ─────────────────────────────                                         │
│   • Unit tests → Function-level properties                              │
│   • Integration tests → Cross-contract properties                       │
│   • Fuzz tests → Edge cases and bounds                                  │
│   • Invariant tests → Direct invariant candidates!                      │
│                                                                          │
│   STEP 3: Extract Properties                                            │
│   ─────────────────────────────                                         │
│   From each test, ask:                                                  │
│   • What assertion is being made?                                       │
│   • What's the implicit invariant?                                      │
│   • What attack is being prevented?                                     │
│                                                                          │
│   STEP 4: Note What's MISSING                                           │
│   ─────────────────────────────                                         │
│   • What functions have no tests?                                       │
│   • What state combinations aren't covered?                             │
│   • What edge cases are untested?                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Test Mining Extraction Template

```markdown
### From Test: [test_function_name]

**Test File:** [path/to/test.sol]
**Test Description:** [what the test does]

**Extracted Property:**
> "[Plain English property derived from test]"

**Test Assert:** `assert(condition)` or `expect(...).to.equal(...)`

**Property Type:** [Valid State / Transition / System-Level / Threat-Driven]

**Coverage Gap?** [Does this test cover edge cases? What's missing?]
```

### What to Extract from Each Test Type

| Test Type | Look For | Extract As |
|-----------|----------|------------|
| `test_deposit_increases_balance` | Postcondition assert | State Transition property |
| `test_cannot_withdraw_more_than_balance` | Revert condition | SHOULD NEVER property |
| `testFuzz_deposit(uint256 amount)` | Bounds, overflows | Valid State + bounds |
| `invariant_totalSupply` | Direct invariant | System-Level property |
| `test_onlyOwner_can_pause` | Access control | Access Control property |
| `test_reentrancy_protection` | Attack prevention | Threat-Driven property |

### Mining Commands

```bash
# Find all test files
find . -name "*.t.sol" -o -name "*.test.js" -o -name "*_test.go"

# Find all assertions in Foundry tests
grep -rn "assert\|assertTrue\|assertEq\|assertGt\|assertLt" test/

# Find all revert expectations
grep -rn "vm.expectRevert\|revert\|require" test/

# Find invariant tests (Foundry)
grep -rn "function invariant_" test/

# Find fuzz tests (Foundry)  
grep -rn "function testFuzz_\|function test.*Fuzz" test/
```

### Blind Spot Analysis

After mining tests, document what's **NOT** tested:

```markdown
## Test Coverage Blind Spots

### Functions Without Tests
- [ ] `emergencyWithdraw()` - no test coverage
- [ ] `setConfig()` - only happy path tested

### State Combinations Untested
- [ ] User with zero balance calling withdraw
- [ ] Multiple users interacting simultaneously
- [ ] State after failed transaction

### Edge Cases Missing
- [ ] Max uint256 values
- [ ] Zero address interactions
- [ ] Empty arrays

### Attack Vectors Unconsidered
- [ ] Flash loan attacks
- [ ] Sandwich attacks
- [ ] Governance manipulation
```

---

## Integration: Complete Property Discovery Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   PHASE 2: COMPREHENSIVE PROPERTY DISCOVERY                             │
│                                                                          │
│   ┌─────────────────┐                                                   │
│   │ 1. Mine Tests   │ ← Extract existing developer assumptions          │
│   └────────┬────────┘                                                   │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │ 2. RIGHT/WRONG  │ ← Dual mindset enumeration                        │
│   │    Checklist    │   (Should always / Should never)                  │
│   └────────┬────────┘                                                   │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │ 3. Categorize   │ ← Valid State / Transition / System / Threat      │
│   │    Properties   │                                                   │
│   └────────┬────────┘                                                   │
│            ▼                                                             │
│   ┌─────────────────┐                                                   │
│   │ 4. Document     │ ← {contract}_candidate_properties.md              │
│   │    All          │   with IDs, owners, external deps                 │
│   └─────────────────┘                                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Updated Quick Checklist

Before leaving Phase 2:

**Standard Checklist:**
- [ ] Every property has a unique ID
- [ ] Every property has plain English description
- [ ] Every property has impact assessment
- [ ] Every property is categorized
- [ ] Variables and owners documented
- [ ] External dependencies flagged
- [ ] Aggregate requirements flagged
- [ ] Trusted role assumptions marked OUT OF SCOPE
- [ ] No CVL syntax anywhere

**NEW - Dual Mindset Checklist:**
- [ ] Each critical function has SHOULD ALWAYS property
- [ ] Each critical function has SHOULD NEVER property
- [ ] Attack vectors explicitly enumerated
- [ ] Attacker goals documented

**NEW - Test Mining Checklist:**
- [ ] Existing tests reviewed
- [ ] Implicit invariants extracted
- [ ] Threat assumptions documented
- [ ] Blind spots identified
- [ ] Coverage gaps noted
---

# 7. PROPERTY PRIORITIZATION

> **Source:** Certora Tutorial - Auction Demonstration

After discovering and categorizing properties, **prioritize them by impact** to focus verification effort.

## 7.1 Priority Levels

| Priority | Criteria | Impact | Example Properties |
|----------|----------|--------|-------------------|
| **HIGH** | Loss of funds | Direct financial loss | "Winner cannot claim prize" |
| **HIGH** | Value creation | Mint without backing | "Mint without collateral" |
| **HIGH** | Privilege escalation | Critical access control | "Non-admin can pause system" |
| **HIGH** | DoS / Griefing | System unavailability | "Auction cannot be resolved" |
| **MEDIUM** | Solvency (when no withdrawal) | Accounting integrity | "Total supply tracking" |
| **MEDIUM** | Temporary inconsistency | Eventually consistent | "Stale price oracle" |
| **LOW** | Single function correctness | Local behavior | "mint() increases balance" |
| **LOW** | Easy to manually verify | Simple arithmetic | "1 + 1 = 2 checks" |

## 7.2 Prioritization Template

For each property, document:

```markdown
### [Priority Level] - [Property ID]: [Property Name]

**Impact:** [Loss of funds / DoS / Manipulation / Accounting / Local behavior]

**Attack Vector:** [How would an attacker exploit the violation?]

**Reasoning:** [Why this priority level?]

**Dependencies:** [Which other properties must hold for this to matter?]

**Verification Cost:** [Simple / Medium / Complex - affects ordering within priority tier]
```

### Example: HIGH Priority

```markdown
### HIGH - VS2: Winner Can Always Claim Prize

**Impact:** Loss of funds

**Attack Vector:** 
If winner cannot claim prize, funds locked forever → direct user loss.
Admin might drain while prize locked.

**Reasoning:** 
Directly affects user funds. Violates core auction promise.

**Dependencies:** 
- VS1 (valid winner selected) must hold
- If no valid winner, this property N/A

**Verification Cost:** Medium (requires preserved block with state setup)
```

### Example: MEDIUM Priority

```markdown
### MEDIUM - VS4: Solvency (Prize Funds)

**Impact:** Accounting integrity

**Attack Vector:** 
If contract balance < prize amount, winner cannot withdraw.
But no withdrawal exists in this contract, so this is accounting-only.

**Reasoning:** 
Important for system integrity but not directly exploitable given current interface.

**Dependencies:** 
- HIGH properties must pass first
- Becomes HIGH if withdrawal mechanism added

**Verification Cost:** Simple (ghost-based sum check)
```

### Example: LOW Priority

```markdown
### LOW - UT1: Bid Function Updates Highest Bidder

**Impact:** Local function behavior

**Attack Vector:** None - this is unit-level correctness

**Reasoning:** 
Verifies single function works correctly. Easy to test manually.
Covered by higher-level properties (VS1, HLP1).

**Dependencies:** None

**Verification Cost:** Simple (before/after check)
```

## 7.3 Practical Priority Assignment

### Step 1: Ask Impact Questions

For each property, ask:
1. **What's the worst outcome if this property is violated?**
   - Loss of funds → HIGH
   - System unusable → HIGH
   - Accounting error → MEDIUM
   - Single function wrong → LOW

2. **Can an attacker exploit this directly?**
   - Yes + financial gain → HIGH
   - Yes + DoS → HIGH
   - No → MEDIUM or LOW

3. **Is there a workaround or mitigation?**
   - No workaround → Increase priority
   - Easy workaround → Decrease priority

### Step 2: Consider Verification Effort

Within same priority level, order by:
- Simple properties first (quick wins)
- Complex properties later (when you understand system better)

### Step 3: Document Trade-offs

```markdown
## Priority Assignment Trade-offs

### Why VS2 is HIGH but VS4 is MEDIUM:
VS2 affects user funds directly (withdrawal blocked).
VS4 is accounting-only since no withdrawal mechanism exists.
If withdrawal added later, VS4 becomes HIGH.

### Why UT1 is LOW:
Single function behavior is covered by:
- VS1 (valid winner) - requires bid() to work
- HLP1 (highest bidder wins) - implies bid() correctness
Redundant verification, keep LOW unless others fail.
```

## 7.4 Integration with Categorization

Your properties should now have:
1. **Category** (Valid State / Variable Transition / High-Level / Unit Test)
2. **Dual Mindset** ("Should Always..." / "Should Never...")
3. **Priority** (HIGH / MEDIUM / LOW)

### Complete Property Entry

```markdown
### VS2: Winner Can Always Claim Prize

**Category:** Valid State  
**Priority:** HIGH  
**Should Always:** Winner address can call claimPrize() successfully after auction ends  
**Should Never:** Winner should never be unable to claim prize due to contract state

**Impact:** Loss of funds  
**Attack Vector:** Funds locked if winner cannot withdraw  
**Reasoning:** Direct user fund loss violates core protocol guarantee

**Plain English:**
After auction ends, the winner address must be able to successfully call claimPrize() 
and receive the prize amount. The contract must hold sufficient funds.

**Verification Cost:** Medium (requires state setup in preserved block)
```

## 7.5 Verification Order Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ROUND 1: HIGH Priority Properties                                  │
│  ├── Simple HIGH properties (quick validation)                      │
│  ├── Complex HIGH properties (critical but time-intensive)          │
│  └── GOAL: Prove no critical vulnerabilities                        │
│                                                                      │
│  ↓ (If all HIGH pass)                                               │
│                                                                      │
│  ROUND 2: MEDIUM Priority Properties                                │
│  ├── Accounting properties                                          │
│  ├── Consistency properties                                         │
│  └── GOAL: Prove system integrity                                   │
│                                                                      │
│  ↓ (If time permits)                                                │
│                                                                      │
│  ROUND 3: LOW Priority Properties                                   │
│  ├── Unit test equivalents                                          │
│  └── GOAL: Comprehensive coverage                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Time-Boxed Strategy

If verification time is limited:
1. **Must verify:** All HIGH priority properties
2. **Should verify:** MEDIUM priorities related to HIGH properties
3. **Nice to have:** All MEDIUM + selected LOW properties

```markdown
## Verification Time Budget (Example)

**Available:** 2 weeks

**Allocation:**
- Week 1: HIGH priority (5 properties) - MUST COMPLETE
- Week 2 Days 1-3: MEDIUM priority (3 properties) - SHOULD COMPLETE  
- Week 2 Days 4-5: LOW priority (7 properties) - BEST EFFORT

**Contingency:**
If HIGH properties reveal bugs, extend Week 1, compress Week 2.
```

---

## Updated Quick Checklist (v1.5)

Before leaving Phase 2:

**Standard Checklist:**
- [ ] Every property has a unique ID
- [ ] Every property has plain English description
- [ ] Every property has impact assessment
- [ ] Every property is categorized
- [ ] Variables and owners documented
- [ ] External dependencies flagged
- [ ] Aggregate requirements flagged
- [ ] Trusted role assumptions marked OUT OF SCOPE
- [ ] No CVL syntax anywhere

**Dual Mindset Checklist (v1.2):**
- [ ] Each critical function has SHOULD ALWAYS property
- [ ] Each critical function has SHOULD NEVER property
- [ ] Attack vectors explicitly enumerated
- [ ] Attacker goals documented

**Test Mining Checklist (v1.2):**
- [ ] Existing tests reviewed
- [ ] Implicit invariants extracted
- [ ] Threat assumptions documented
- [ ] Blind spots identified
- [ ] Coverage gaps noted

**NEW - Prioritization Checklist (v1.3):**
- [ ] Each property assigned priority (HIGH / MEDIUM / LOW)
- [ ] Impact documented for each property
- [ ] Attack vectors identified
- [ ] Dependencies between properties noted
- [ ] Verification cost estimated
- [ ] Priority trade-offs explained
- [ ] Verification time budget allocated