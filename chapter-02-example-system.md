# Chapter 2: The System We Are Building

## Why This Example Exists

The workflow described in Chapter 1 is abstract until you apply it to something concrete. This chapter introduces the example system that the rest of the book builds against: a small retirement-planning projection engine, deliberately scoped to be teachable while still complex enough to force the discipline.

The system is small. One person, two accounts, two budget categories, a 35-year horizon. You could sketch it on a napkin. But it has a property that makes it a perfect vehicle for what this book is about: a single rule, required minimum distributions, creates cross-cutting constraints that ripple through the entire projection loop. Without explicit specification, that rule admits multiple plausible implementations. An LLM will produce any of them confidently, and most of them will be subtly wrong.

The goal of this chapter is not to build the system. It is to define it precisely enough that every subsequent chapter can reference it: when we write specs in Chapter 3, we are writing specs for this system. When we run deterministic validation in Chapter 8, we are validating this system's arithmetic. When we add new requirements in Chapter 15, they land on this system.

## The Minimal Model

The projection models a single person over a 35-year horizon in annual steps. Each year, the engine applies returns, processes withdrawals, funds spending, and advances the state. The person has two accounts and two spending categories.

**Two accounts:**

1. **Liquid account.** A taxable cash or brokerage-like account. Money in, money out, earns a return. No withdrawal restrictions.
2. **401(k) account.** A tax-deferred retirement account. Money grows without taxation, but withdrawals are subject to rules. Specifically: once the person reaches a certain age, the IRS requires a minimum annual distribution. That is the RMD rule, and it is the centerpiece of this example.

**Two budget items:**

1. **Essential spending.** Non-discretionary costs: housing, food, insurance, the expenses that do not go away. The engine funds these first.
2. **Discretionary spending.** Everything else: travel, dining, hobbies. The engine funds these second, after essentials.

**Withdrawal policy (baseline):**

Spending is funded from the liquid account first. If the liquid account cannot cover the year's spending, the shortfall is withdrawn from the 401(k). This is a simple waterfall: liquid first, then 401(k). No other accounts, no other rules, no rebalancing. Just two buckets and a priority order.

**Two engines:**

1. **Deterministic.** A single projection with fixed return rates. The same inputs always produce the same outputs.
2. **Monte Carlo.** Multiple projections (paths) with randomized returns drawn from a distribution. Seeded for reproducibility: given the same seed, the same paths are generated every time.

Both engines run the same annual loop. The only difference is how returns are generated. The RMD rule, the withdrawal policy, the spending priority: all of it is identical across both engines. This is not optional. If the deterministic engine applies RMD before spending and the Monte Carlo engine applies it after, you have two systems, not one.

## Where RMD Fits in the Annual Loop

Each year of the projection follows a sequence. The order matters.

```
For each year in the 35-year horizon:

  1. Apply returns to both accounts (growth for the year)
  2. Check RMD obligation
     - If the person has reached the RMD start age:
       compute RMD = prior_year_end_401k_balance / divisor(age)
     - Withdraw the RMD amount from the 401(k)
     - Deposit the RMD amount into the liquid account
  3. Fund essential spending from the liquid account
  4. Fund discretionary spending from the liquid account
  5. If liquid account cannot cover spending:
     - Withdraw the shortfall from the 401(k)
  6. Record end-of-year balances for both accounts
```

Step 2 is the RMD engine's jurisdiction. It runs before spending decisions. This ordering is a design choice, and it has consequences: the RMD distribution lands in the liquid account before spending draws from it. That means the liquid account has more cash available for spending in years when RMD applies. If you reversed the order and processed spending first, the 401(k) might get hit twice in the same year: once for spending shortfalls, once for RMD. The ordering eliminates that ambiguity.

## The RMD Rule

The IRS requires that once a person reaches a certain age, they must withdraw a minimum amount each year from their tax-deferred retirement accounts. The minimum is computed by dividing the prior year's ending balance by a divisor that decreases with age. As the person gets older, the divisor gets smaller, which means the required distribution gets larger as a fraction of the balance.

The formula:

```
RMD = prior_year_end_401k_balance / divisor(age)
```

The divisor comes from the IRS Uniform Lifetime Table (Publication 590-B). For this example, we use a small illustrative subset. This is a toy table for pedagogy, not a regulatory implementation:

| Age | Divisor |
|-----|---------|
| 73  | 26.5    |
| 74  | 25.5    |
| 75  | 24.6    |
| 76  | 23.7    |
| 77  | 22.9    |
| 78  | 22.0    |
| 79  | 21.1    |
| 80  | 20.2    |

The RMD start age is configurable. We treat it as a constant for any given projection. In current IRS rules it is 73 (under SECURE 2.0). The point is engineering, not policy history. If the start age changes next year, you change the constant. The engine does not care.

### A Concrete Example

A person turns 75 this year. Their 401(k) ended last year at $500,000. The divisor for age 75 is 24.6.

```
RMD = $500,000 / 24.6 = $20,325.20
```

The engine withdraws $20,325.20 from the 401(k) and deposits it into the liquid account. If the person's essential spending is $40,000 and discretionary spending is $15,000, the liquid account needs $55,000 total. The RMD contributed $20,325.20 toward that. The liquid account covers the rest from its existing balance. If the liquid account still comes up short, the engine goes back to the 401(k) for the remainder.

### A Second Example

Same person, same age, but their 401(k) ended last year at $10,000. The RMD is:

```
RMD = $10,000 / 24.6 = $406.50
```

Smaller balance, same divisor, much smaller distribution. The liquid account gets $406.50. If that is not enough to cover spending, the 401(k) gets hit again in step 5 for the shortfall.

## The Edge Cases That Define the System

Five behavioral questions must be answered before the first line of code is written. Each of these becomes an invariant later. Right now, they are design decisions.

### 1. Ordering: RMD before spending

The RMD distribution is processed before spending withdrawals. The RMD amount lands in the liquid account first, then spending draws from the liquid account. This is not the only valid ordering, but it is the one we are specifying. Any implementation that processes spending before RMD is wrong for this system.

### 2. Excess RMD

If the RMD is larger than the year's spending need, the excess stays in the liquid account. The person still owns the money. It is not lost, not consumed, not reinvested into the 401(k). It sits in the liquid account and earns the liquid account's return rate in subsequent years.

### 3. Insufficient 401(k) balance

If the 401(k) balance is less than the computed RMD, the engine withdraws whatever is left. The 401(k) goes to zero. The shortfall between the required RMD and the actual withdrawal is recorded. Whether this constitutes a projection failure depends on the system's success definition, which is specified elsewhere. For now: withdraw what you can, record the gap.

### 4. No negative balances

The liquid account cannot go negative. If spending exceeds the liquid account's balance plus whatever can be pulled from the 401(k), spending is curtailed. The engine does not borrow. The engine does not create money. If both accounts are depleted and spending cannot be funded, that is a failure condition.

### 5. Identical rules across engines

The RMD rule, the withdrawal waterfall, the spending priority, and the no-negative-balance constraint all apply identically in the deterministic engine and in every Monte Carlo path. The divisor lookup is deterministic: age in, divisor out. The only thing that varies across Monte Carlo paths is the return applied in step 1. Everything after returns are applied is the same logic.

## The Invariants

These five decisions produce testable invariants. Each one is a statement that must hold for every year in every projection, deterministic or Monte Carlo:

```
INV-1: RMD is processed before spending in every year.

INV-2: RMD excess is deposited into the liquid account, not discarded.

INV-3: If 401(k) balance < computed RMD, the engine withdraws the full
       remaining balance and records the shortfall.

INV-4: Neither account balance may be negative at the end of any year.

INV-5: The annual loop logic (steps 1-6) is identical in the deterministic
       engine and in every Monte Carlo path. Only return generation differs.
```

These are not aspirational. They are contracts. A correct implementation satisfies all five in every year of every projection. A test suite derived from these invariants will catch any implementation that violates them, regardless of whether the violation is obvious or subtle.

## What Can Go Wrong

If you ask an LLM to implement a retirement projection with RMD support and do not specify these invariants, here is what you will get:

**Ordering ambiguity.** The LLM will pick an order for RMD and spending. It might process spending first, then RMD. It might interleave them. It might process RMD and spending as a single combined withdrawal. All of these are reasonable interpretations. None of them match what we specified. If the test suite does not check ordering, the mismatch is invisible.

**Silent excess absorption.** The LLM might treat the RMD as spending rather than as a transfer. If the RMD exceeds spending, the excess vanishes instead of landing in the liquid account. The projection runs. The numbers look plausible. But the person's liquid balance is wrong in every year after the first RMD, and the error compounds.

**Negative balance tolerance.** Without an explicit no-negative-balance constraint, the LLM might allow an account to go slightly negative and correct it the next year. Or it might floor at zero but not record the shortfall. Or it might halt the projection entirely. Each of these is a different system with different behavior. The spec picks one.

**Engine divergence.** The most dangerous failure. The LLM implements the deterministic engine and the Monte Carlo engine as separate classes with separate logic. The deterministic engine processes RMD first. The Monte Carlo engine, written in a different session or a different prompt, processes spending first. Both pass their own tests. The integration test that would catch the divergence does not exist because nobody specified INV-5 as a testable contract.

None of these failures are bugs in the traditional sense. The code compiles. The tests pass. The outputs look reasonable. The system is wrong in ways that only surface when you compare its behavior against an explicit specification of what it should do. That is the gap this book addresses.

## What Comes Next

Chapter 3 introduces spec writing: the discipline of turning the decisions in this chapter into formal behavioral contracts. The RMD engine, the withdrawal waterfall, the ordering rule, the invariants, all of it gets expressed in the spec format that every downstream mode in the pipeline operates against.

The example stays small on purpose. Two accounts, two budget items, one cross-cutting rule. Complex enough to force the discipline. Simple enough to fit in your head. When the pipeline runs against this system, you will see exactly where each mode adds value, because the system is small enough that you can trace the flow from spec to test to code to audit without getting lost.

Later chapters will extend the system. Chapter 15 adds new requirements. Chapter 16 shows what happens when you inherit existing code and need to build specs around it. But the core stays the same: one person, two accounts, and a rule from the IRS that creates more engineering complexity than its two-line formula suggests.
