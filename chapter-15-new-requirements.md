# Chapter 15: New Requirements

## The Phone Call

You have a system. It works. The specs are frozen. The tests pass. The fidelity audit is clean. The traceability matrix maps every requirement to code. You are, by any reasonable measure, done.

Then a new requirement arrives.

This is not a hypothetical. If you have shipped software professionally, you know exactly how this goes. The requirement is reasonable. It is clearly needed. It was not in the original scope because nobody thought of it, or because it was explicitly deferred, or because the business context changed after you started building. The reason does not matter. What matters is that your frozen, validated, tested, audited system now needs to change.

The question is not whether this will happen. It is whether your process can absorb it without losing the properties you spent twelve modes establishing.

## The Requirement

The example system from Chapter 2 models a single person with two accounts (liquid and 401(k)), two spending categories, and an RMD rule. The projection runs for 35 years. Both the deterministic and Monte Carlo engines use the same annual loop. Five invariants govern the system's behavior.

One thing the system does not model: taxes.

When the person withdraws $10,000 from their 401(k), the engine treats that as $10,000 of spending power. In reality, 401(k) withdrawals are taxable income. A $10,000 withdrawal at a 22% effective rate only provides $7,800 of usable cash. The other $2,200 goes to taxes. The system ignores this, and the projection is optimistic by the exact amount of the tax liability.

The new requirement:

> Withdrawals from the tax-deferred account should reflect the fact that taxes will be owed on those withdrawals.

This is a single sentence. It will touch every part of the system.

## Why Not a Full Tax Engine

The instinct, yours and the LLM's, is to build a real tax model. Brackets. Deductions. Filing status. Capital gains rates. State taxes. The domain is well-defined, the IRS publishes the rules, and the LLM will happily produce a complete tax calculator if you let it.

Do not let it.

The purpose of this example is not to model taxes accurately. It is to demonstrate what happens when a new cross-cutting requirement lands on a system that was not designed for it. A flat tax proxy serves that purpose. A full tax engine obscures it behind domain complexity that has nothing to do with the engineering lesson.

The scope decision is explicit: a single flat effective tax rate, applied only to 401(k) withdrawals. No brackets, no deductions, no filing status, no capital gains, no state taxes, no credits, no phase-outs, no inflation adjustments. If the LLM suggests any of these, the answer is "out of scope." If you find yourself debating whether to add capital gains treatment, you have already lost the thread. The engineering lesson is about process, not tax policy.

This is the first test of the discipline. Scope control under pressure. New requirements invite scope creep, and LLMs accelerate scope creep because they can produce plausible implementations of features you did not ask for. The flat tax proxy is a deliberate constraint: complex enough to ripple through the system, simple enough to reason about completely.

## The Flat Tax Proxy

Define a single configurable value:

```
tax_rate = 0.22
```

Every dollar withdrawn from the 401(k) incurs tax at this rate. The rule:

```
tax = gross_withdrawal * tax_rate
net_cash = gross_withdrawal - tax
net_cash = gross_withdrawal * (1 - tax_rate)
```

Before this change, the system had one concept: withdrawal amount. After this change, the system has three: gross withdrawal, tax paid, and net cash. Every place in the engine that uses a 401(k) withdrawal amount must now distinguish between these three values. "Withdrawal" is no longer a single number. It is a triple.

This is why a one-sentence requirement touches everything.

## What Changes in the Annual Loop

The annual loop from Chapter 2 had six steps. Steps 2 and 5 both involve 401(k) withdrawals. Both need to change.

**Step 2: RMD processing.**

The RMD formula is unchanged: `RMD = prior_year_end_401k_balance / divisor(age)`. But the RMD is a gross amount. The IRS requires you to withdraw $20,325.20 from your 401(k). You withdraw that amount. Then you owe taxes on it.

```
RMD gross = $20,325.20
Tax = $20,325.20 * 0.22 = $4,471.54
Net cash = $20,325.20 - $4,471.54 = $15,853.66
```

The liquid account receives $15,853.66, not $20,325.20. The difference goes to taxes. The 401(k) balance decreases by the full gross amount.

**Step 5: Spending shortfall withdrawals.**

If the liquid account cannot cover spending, the engine withdraws the shortfall from the 401(k). But now the withdrawal must be grossed up, because the person needs $10,000 of net spending power, not $10,000 of gross withdrawal.

```
Required net: $10,000
Gross withdrawal: $10,000 / (1 - 0.22) = $12,820.51
Tax: $12,820.51 * 0.22 = $2,820.51
Net cash: $12,820.51 - $2,820.51 = $10,000.00
```

The 401(k) decreases by $12,820.51. The liquid account receives $10,000.00. Taxes consume $2,820.51. Without the gross-up, the person gets $7,800 of spending power from a $10,000 withdrawal and comes up short.

## RMD and Spending Interaction

RMD is still processed before spending (INV-1 from Chapter 2 is unchanged). But the interaction is now more nuanced.

If the RMD provides more net cash than spending requires, the excess stays in the liquid account. This is the same rule as before, but the excess is now after-tax:

```
RMD gross = $15,000
Tax = $15,000 * 0.22 = $3,300
Net cash = $11,700

Spending required = $6,000
Excess to liquid = $11,700 - $6,000 = $5,700
```

If spending requires more than the RMD's net cash, the engine must make an additional 401(k) withdrawal for the shortfall, grossed up:

```
RMD gross = $8,000
Tax on RMD = $1,760
Net from RMD = $6,240

Spending required = $10,000
Shortfall = $10,000 - $6,240 = $3,760
Additional gross withdrawal = $3,760 / 0.78 = $4,820.51
Tax on additional = $1,060.51
```

Total 401(k) reduction: $8,000 + $4,820.51 = $12,820.51. Total tax: $2,820.51. Total net to liquid: $10,000.00. The accounting identity holds.

## The New Invariants

The five invariants from Chapter 2 remain. The tax proxy adds three more:

```
INV-6: For every 401(k) withdrawal, net_cash = gross * (1 - tax_rate).
        No withdrawal violates this identity.

INV-7: When funding a spending shortfall from the 401(k), the gross
        withdrawal is computed as shortfall / (1 - tax_rate). The net
        cash received equals the shortfall exactly.

INV-8: RMD is a gross amount. The minimum distribution requirement is
        satisfied by the gross withdrawal, not the net cash. The engine
        never under-withdraws by confusing gross and net.
```

INV-8 is the subtle one. If the engine computes RMD as a net requirement and then grosses it up, the person withdraws more than the IRS requires. That is not illegal, but it changes the projection: the 401(k) depletes faster, tax payments are higher, and the liquid account accumulates more after-tax cash. The system behaves differently depending on whether RMD is interpreted as gross or net, and both interpretations produce valid-looking projections. Only the spec distinguishes them.

## What the Pipeline Does

This is where the twelve modes earn their keep.

The specs are frozen. The new requirement cannot be implemented against the current spec surface because the current specs say nothing about taxes. The process:

1. **Unfreeze.** Delete the lock file and tag file. The spec surface is now mutable.

2. **Write new specs.** The tax proxy needs its own spec: the rate, the gross-up formula, the three-value withdrawal model. The RMD spec needs revision: clarify that RMD is a gross amount, add the tax interaction. The withdrawal policy spec needs revision: the shortfall calculation now includes gross-up. The annual loop spec needs revision: tax is a new output recorded per year.

3. **Run spec review (Mode 2).** Each revised spec is reviewed for behavioral completeness. The tax proxy spec gets the same treatment as every other spec: behaviors, acceptance tests, out-of-scope section.

4. **Run deep review (Mode 3).** The vocabulary has changed. "Withdrawal" now means three things. Deep review catches any spec that still uses "withdrawal" without qualifying gross or net.

5. **Run coherence (Mode 4).** The tax proxy creates new cross-module handoffs. Coherence traces the data flow: where does gross_withdrawal originate, where does net_cash get consumed, where does tax_paid get recorded? Any gap in the chain is a contract mismatch.

6. **Refreeze (Mode 5).** New tag, new lock file. The spec surface is immutable again, now including the tax proxy.

7. **Selective re-run of Modes 6 through 12.** The chain hash model identifies which specs changed and which downstream artifacts need regeneration. Unchanged specs skip validation. Changed specs get the full treatment: new golden cases for the tax arithmetic, new tests, new implementation, new fidelity audit, updated traceability matrix.

The pipeline does not start over. It re-runs the affected slice. The incremental model, the chain hashes, the manifests that track what was checked and when: all of it exists for this moment. A new requirement is not a crisis. It is a revision cycle, and the pipeline has a well-defined path for it.

## The Scope Discipline

The hardest part of absorbing a new requirement is not the technical work. It is holding the line on scope.

The tax proxy is a flat rate on 401(k) withdrawals. That is the requirement. But the moment you start implementing it, the LLM will suggest related improvements. "Should we also handle capital gains on the liquid account?" No. "What about the standard deduction reducing the effective rate?" No. "Should the tax rate vary by income level?" No. Each suggestion is reasonable. Each one makes the system more realistic. And each one doubles the surface area of the change, introduces new invariants that need testing, and creates new cross-spec dependencies that need coherence checking.

The discipline is: implement the requirement as specified, verify it through the pipeline, and stop. If the flat rate needs to become brackets later, that is a future requirement. It will arrive the same way this one did, and the pipeline will handle it the same way.

## What Can Go Wrong Without the Pipeline

If the system had no frozen specs, no review process, and no formal revision cycle, the tax proxy would be implemented as a code change. The developer (or the LLM) would modify the withdrawal logic, add a tax_rate parameter, adjust the spending shortfall calculation, and call it done.

The tests that existed before the change would still pass, because they do not test for tax behavior. The new behavior would have no tests of its own, because nobody wrote a spec that generates them. The invariant that gross-up produces exact net spending (INV-7) would not be checked, because nobody stated it. The RMD gross-vs-net distinction (INV-8) would be resolved by whichever interpretation the developer happened to use, and that interpretation would be invisible in the code unless someone read it carefully enough to notice.

Six months later, someone asks: "Does RMD mean gross or net in our system?" And the answer is: "Check the code." That is the answer the pipeline replaces with: "Check the spec."
