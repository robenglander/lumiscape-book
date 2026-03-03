# Chapter 3: Spec Writing

Every spec in this book follows a formal structure. Behaviors have sequential IDs. Acceptance tests trace back to those behaviors, and every behavior traces forward to at least one test. Sections appear in a fixed order. Required fields are required. The format is strict because everything downstream depends on it: review checks completeness against the structure, validation checks correctness against the behaviors, test generation maps directly to behavior IDs, traceability links specs to production code. The spec is the source of truth. If the spec is ambiguous, everything built from it inherits that ambiguity.

Writing in this format is not difficult, but it is not where the hard work happens. The hard work is deciding what the behaviors should be. What the edge cases are. What is in scope and what is not. Those are engineering decisions, and they happen in conversation.

You talk to Claude in natural language. You describe what you need, answer questions, make decisions. Claude writes the spec in the formal structure. You can also edit spec files directly, and sometimes that is faster. The conversation is the bridge between what you know about the problem and a spec that captures it precisely enough to build against.

## The Thinking Problem

The distance between a narrative description and a behavioral contract is not a formatting gap. It is a thinking gap.

"The withdrawal calculator handles account withdrawals based on the account type, balance, and applicable penalties." That sentence lets the writer avoid six decisions: which account types, what happens when balance is insufficient, what the penalties are, what the thresholds are, what errors are thrown, and what is out of scope. The writer knows the answers to some of those. The writer has not thought about the rest. The narrative form lets both states coexist invisibly.

The conversational process makes the gap visible. When Claude asks "what happens when balance is insufficient?" and the answer is "I haven't decided," that is useful information. It means this behavior needs design work before it can be specified. If the spec were written as a narrative, the "I haven't decided" would be hidden behind "handles withdrawals appropriately."

Spec writing is the discipline of making decisions before implementation begins. The conversation is one mechanism for surfacing those decisions. Claude asks the questions that force ambiguity into the open. Whether the result ends up in the spec through conversation or through direct editing, what matters is that the decisions are explicit and the behaviors are testable.

## Building Domain Knowledge

Spec writing sometimes starts before you have a complete understanding of requirements. Claude is a useful research partner during spec development.

Say you are building a retirement planning engine. You know there are rules about withdrawing money from retirement accounts. You know there are penalties for early withdrawal. You know there is something called a required minimum distribution. But you do not know the details: the age thresholds, the IRS tables, the interaction between different account types, the tax implications.

Start the conversation:

> "I need to understand the rules for required minimum distributions from traditional IRAs. What are the current IRS rules? Cite the specific IRS publications."

Claude comes back with the rules: RMDs begin at age 73 (under SECURE 2.0 Act), the distribution amount is calculated by dividing the prior year-end balance by a life expectancy factor from IRS Publication 590-B Table III, the penalty for missing an RMD is 25% of the shortfall (reduced from 50% by SECURE 2.0). Claude cites IRS Publication 590-B, the SECURE 2.0 Act of 2022, and the relevant Internal Revenue Code sections.

Guide Claude toward authoritative sources. Government publications, regulatory text, official standards. Not blog posts, not financial advisor websites, not forum answers. When Claude cites an IRS publication, you can verify it. When the source is vague, you cannot. This discipline pays off later: when you run deterministic validation (Chapter 8), you verify your engine's output against hand-calculated values derived from those same primary sources. The authoritative citation from spec writing becomes the ground truth for validation.

> "Show me the actual IRS Uniform Lifetime Table from Publication 590-B. I want the divisors for ages 72 through 85."

Claude produces the table. Now both you and Claude have domain knowledge grounded in a primary source, and the conversation that follows is grounded in the same facts. The IRS Publication 590-B reference ends up in the spec's References section, traceable to the source.

## Using Multiple LLMs

Sometimes you want a different perspective on the domain, or a different model's training data surfaces something Claude missed.

I use ChatGPT and Gemini for domain research regularly. I ask them the same questions I would ask Claude: explain the IRS rules for Roth conversions, what are the income limits, cite the publications. When I find what I need, I bring the results back to Claude as specific instructions:

> "Here are the IRS rules for Roth conversions that I verified against Publication 590-A. Update the Roth conversion calculator spec to reflect these rules: [specific rules with citations]."

These LLMs have no ego. ChatGPT will cheerfully write a detailed brief for Claude. Gemini will produce a structured summary formatted for spec integration. Or do your own research. There's no single best way to do this.

The discipline is the same regardless of which LLM does the research: you should rely on authoritative sources only, citations required, verify before incorporating.

## From Domain Knowledge to Decisions

Once you have the domain knowledge, the conversation shifts from research to requirements. This is where decisions happen.

> "I need a calculator that computes required minimum distributions. It should use the IRS Uniform Lifetime Table divisors."

Claude asks questions:

> "Should the divisor table be injected from configuration, or hardcoded into the calculator?"

Decision: injected. The user might want to model different table versions.

> "What happens if the person is below age 73?"

Decision: return zero. No RMD obligation below that age.

> "What about the edge case where the age is not in the table?"

Decision: throw an exception. That should not happen with valid input.

> "Is the early withdrawal penalty in scope for this calculator?"

Decision: out of scope. A separate calculator handles penalties.

Four questions, four decisions. Claude converts those decisions into spec behaviors with the correct IDs, proper coverage, and the right structure. If you already know the domain well, this phase moves quickly. If you are still building domain knowledge, the conversation naturally weaves research and decision-making together.

## Iterative, Not Linear

The process loops. It is not a straight line from research to finished spec.

Claude produces a draft. You read it and notice something: the spec handles the case where someone turns 73 mid-year, but you never discussed whether the RMD is prorated or applies for the full year. You ask Claude. Claude asks back: "The IRS rules say the first RMD is due by April 1 of the year after you turn 73. Should the calculator model the April 1 deadline, or just the age-based trigger?" You decide: age-based trigger only, the April 1 deadline is an operational concern, not a calculation concern. Claude updates the spec.

Ask Claude to summarize the current state. Read the behavior list. Notice that B-004 and B-005 overlap. Say so. Claude consolidates them.

Ask Claude for feedback: "Are there IRS edge cases I'm missing?" Claude might surface inherited IRAs, which have different distribution rules under the SECURE Act. You did not know that. Now you decide: in scope or out of scope? Either answer is fine. The point is that the question was asked and the decision was made explicitly.

Ask Claude for recommendations: "Should the divisor table be a separate DTO or embedded in the scenario?" Claude has opinions about this, informed by the existing architecture. Its recommendation might be good. Take it or reject it. You would be surprised how helpful Claude can be on architectural questions when it has enough context. Just make sure you are evaluating its recommendation, not rubber-stamping it. The difference between "Claude recommended this and I agreed" and "Claude did this and I didn't notice" is the difference between a decision and an assumption.

The iteration crosses session boundaries too. You write a spec today. Next week, while working on a different spec, you realize the first one is missing a behavior. You come back to it. Same process.

## What the Conversation Prevents

If you skip the conversation and say "write me an RMD calculator spec," Claude will produce a complete spec. It will be structurally sound. It will have behaviors, tests, metadata. It will look correct. But Claude will have made every decision itself, filling gaps with reasonable assumptions. The divisor table will be hardcoded because that is simpler. The early withdrawal penalty will be in scope because it is related. The edge case for inherited IRAs will be handled with a default rule because leaving it unspecified feels incomplete. Every one of those decisions might be wrong, and you will not notice because the spec looks thorough.

The conversation makes decisions explicit. Claude asks instead of assuming. You decide instead of discovering problems after the fact.

There is a subtler version of the same problem. Even with the conversation, you can accidentally delegate by being passive. Claude asks "should the divisor table be hardcoded?" and you say "sure, whatever you think." Claude's answer might be right, but you have not evaluated it. When the IRS updates Publication 590-B and the divisor table needs to change, the hardcoded table will be wrong, and the spec will not have anticipated it, because the question of configurability was never actually decided.

## Why the Format Matters

The spec format is strict. Every spec follows a structural template with thirteen ordered positions:

---
**Structural template positions (summary):**

```
POS  SECTION                         REQUIRED?
---  -------                         ---------
1    Header Block                    REQUIRED
2    Architecture Metadata           REQUIRED
3    Architecture Diagrams           CONDITIONAL
4    Purpose                         REQUIRED
5    Location / File Organization    REQUIRED
6    FLEX ZONE                       CONDITIONAL
7    Design Decisions                CONDITIONAL
8    Edge Cases                      CONDITIONAL
9    Out of Scope (Non-Goals)        REQUIRED
10   Core Behaviors                  REQUIRED
11   Acceptance Tests                REQUIRED
12   References                      REQUIRED
13   Changelog                       CONDITIONAL
```
---

It makes specs greppable. Every behavior has a `B-NNN` ID. Every acceptance test has an `AT-NNN` ID. When you search across a hundred specs for a specific behavior, you get a hit with a file, a line number, and an ID you can trace.

It makes specs mechanically validatable. Structural checks verify that every required section exists, that IDs are sequential, and that every behavior is covered by at least one test and every test traces to at least one behavior. That bidirectional coverage rule is the most important structural property.

It makes specs navigable. Core Behaviors are always in the same position. References are always last.

The domain content lives in position 6, the FLEX ZONE. Its internal structure varies by component type: algorithms and tables for calculators, schema and interfaces for repositories, endpoints and configuration for services. The FLEX ZONE is where the IRS divisor table goes, where the tax bracket data goes, where the algorithm pseudocode goes. It is the substance of the spec, drawn from the domain knowledge you built during the conversation.

Claude knows the format and applies it. If you prefer to learn it yourself, the full template is in the project's system specs. Chapter 5 covers the structural validation that enforces it.

## What Comes Next

Chapter 4 covers `/spec-review`, which validates specs for behavioral completeness. Review presupposes a structurally valid spec: Core Behaviors, Out of Scope, acceptance tests. The conversation naturally produces the structure, because Claude writes the spec in the correct format from the start. What review adds is a second pass: are the behaviors complete? Are the acceptance tests thorough? Are there edge cases the conversation missed?
