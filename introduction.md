# Introduction

Large language models can write code. The industry is very excited about this. Benchmarks measure how fast models generate functions. Product launches trumpet lines-of-code-per-hour improvements. Engineering blogs celebrate the productivity gains from AI-assisted coding.

They are optimizing the wrong thing.

In software development, writing code is one part of the work. It is not the biggest part. If you have built and shipped a system of any complexity, you already know this. The time goes to understanding requirements. Making design decisions. Catching the contradiction between what module A expects and what module B provides. Validating that your tax calculation actually matches the IRS publication you think it matches. Tracing a requirement through its test, its implementation, and its production behavior. Maintaining consistency across a codebase that is too large to hold in your head.

That is the engineering. The code is what falls out of it.

An LLM that writes code faster saves you time on the smallest part of the job. An LLM that does engineering, that reviews specs for completeness, validates calculations against authoritative sources, traces data flows across module boundaries, audits implementations for fidelity, catches cross-spec contradictions before they become cross-module bugs, saves you time on the work that actually dominates.

This book is about the second kind.

---

## What This Book Is Not

If you are fixing a bug, writing a utility method, or adding a small feature to a codebase you already own, go ahead and use an LLM to write the code. For small tasks, coding genuinely is most of the work. The LLM is the right tool and you will be fine. This book is not about that.

This book is for projects where the engineering surface is large enough that the non-coding work dominates. If you are building a system with interacting components, domain-specific correctness requirements, and enough moving parts that you cannot hold it all in your head, then writing code is maybe 20% of what you need to get right. The other 80%, the requirements, the design decisions, the validation, the consistency, the traceability, is where this book lives.

If you are in the middle of a large project and realize it lacks this kind of structure, that is also relevant. But the starting point is recognizing whether the engineering surface warrants it. This book assumes it does.

This is not an attempt to invent a new engineering methodology. Behavioral contracts, explicit specifications, acceptance tests, traceability matrices, and test-driven development have a long history. Engineers have done this for decades. What is different is the context: applying that discipline with a generative model as the execution engine, not just for code, but for the engineering itself. The discipline is established. The application is new.

---

## The System That Forced the Discipline

The system I'm building throughout this book is part of a retirement planning engine. Not a demo. Not a toy project. A real engine I intend to rely on.

It models deterministic projections. It runs Monte Carlo simulations. It handles tax rules, required minimum distributions, regime switching, fat tails, portfolio sequencing, and validation across multiple interacting components. It's the kind of system where being "mostly correct" is not good enough.

I wrote it in Java. I use Maven. Those are my choices. They are not prescriptions. You could apply everything in this book in Kotlin, Go, Rust, C#, whatever you work in. The language is not the point. The discipline is.

This isn't scaffolding for a methodology. The methodology showed up because the system forced it.


## How This Started

This didn't begin as a book idea.

It began because I wanted to understand my own retirement plan.

After more than four decades building and leading software systems, I found myself wanting answers about how black-box tools were deriving results. I wanted to see the math. I wanted to see the assumptions. I wanted to run scenarios and understand why results changed.

So I started building my own engine.

At some point, I pulled an LLM into the workflow. The productivity jump was immediate. Refactors became trivial. Documentation could be generated on demand. Acceptance tests could be drafted in minutes.

And then something subtle started happening.

I would review a spec and find that a small assumption had been filled in. A validation rule implied but not stated would quietly disappear. An ambiguity would be resolved in a way that felt reasonable, but wasn't what I meant.

Nothing was obviously broken. But correctness became probabilistic.

A gap in the spec is not neutral. It is a license for the model to fill intent with plausible inference. The inferences are frequently wrong in ways that are invisible until they have been compiled into artifacts and validated against tests derived from the same inference.

I caught one case where a behavior I hadn't specified showed up in generated tests anyway. The tests passed. The code compiled. Everything looked fine. But the behavior was invented. It was logical. It was clean. And it was wrong.

That was one turning point. But it was not the important one.

The important realization came later: I was using the LLM to write code, and the code was the easy part. The hard part was everything around it. Reviewing specs for completeness. Catching inconsistencies between specs that were written weeks apart. Validating my RMD calculations against IRS publications. Making sure the deterministic engine and the Monte Carlo engine agreed on what "success" means. Tracing every requirement through its test and its implementation.

I was doing all of that by hand. The LLM was writing code for me, and I was spending my time on engineering.

So I flipped it. I started building skills, structured instruction sets, that pointed the LLM at the engineering work instead.

---

## The Twelve Modes

The breakthrough wasn't a better prompt. It was a shift in what the LLM does.

Instead of asking the LLM to write software and then reviewing the code, I moved the center of gravity upward. The spec became the authoritative source. Everything downstream became compiled artifacts: code, tests, documentation, diagrams. And the LLM's primary job became engineering those artifacts, not just generating code.

I built a twelve-mode pipeline. Eleven of the twelve modes are engineering work. One writes production code. That ratio is not an accident. It reflects where the actual work is:

- Three modes review specs for individual completeness, cross-spec consistency, and behavioral coherence.
- One mode freezes the spec surface so everything downstream builds against a stable target.
- Three modes validate the specs against external authority: IRS publications, mathematical properties, semantic alignment between engines.
- One mode generates the complete test suite before any production code is written.
- One mode implements production code against those failing tests.
- One mode audits the implementation for fidelity, completeness, and containment.
- One mode maps every requirement to its test, its code, and its authoritative source.

The model does all of this. Not just the coding part. All of it. Reviewing, validating, tracing, auditing. The engineering.

One specific problem needed a dedicated rule. The model has no persistent cross-session state. A decision made in yesterday's session does not exist in today's context window. An architectural constraint established two weeks ago is invisible unless it is embedded in a file the model reads right now. Left unaddressed, the model will assert things about the spec surface based on vague recollection rather than current file contents. The fix is structural: every behavioral claim must be grounded in a tool call from the current session. Not memory. Not inference from prior conversations. A concrete tool call: grep the file, read the section, show the evidence. This is the NO-MEMORY RULE, and it applies to every skill in the pipeline without exception.

I began insisting on repeated clean review passes before freezing anything. If the surface moved, it wasn't stable. If it wasn't stable, it wasn't ready. That simple discipline changed everything.

---

## The Engineer Remains the Engineer

Applying LLMs to engineering work does not change who is responsible.

On my retirement planning engine project I am the architect. I define the module boundaries, the behavioral contracts, the dependency structure, and the acceptance criteria. The model generates artifacts based on those contracts: specs, tests, code, diagrams, audit reports. It does not decide what to build. It builds what is specified. And if the outcome is wrong, that's my fault. There's no tolerance for abdicating responsibility.

Nothing in this workflow executes without explicit authorization. The model proposes an action, explains what it will do and why, and waits. I review it. I approve it or I don't. There is no sweeping approval. There is no "run everything." Each action is deliberate.

That deliberation happens in conversation. You invoke a mode, Claude runs it, reports its findings, and the exchange continues from there: findings, approvals, directions, decisions. The execution medium is unchanged from the ad-hoc LLM usage you already know. What changed is the grounding. The conversation operates against a spec surface you control, with constraints loaded from files, not stated in prompts and forgotten.

None of this makes the model reliable. It will still make mistakes. It will skip a step in a skill and not mention it. It will assert something about a spec it has not actually read in this session. It will produce a plausible-looking output that missed the point. The structure catches many of these: the evidence requirement in the manifest, the freeze gate, the no-memory rule. But it does not catch all of them.

That is still your job. If something looks off, stop it. Question the evidence. Ask it to re-read the section. When a result seems too clean or arrives too fast, investigate. And when the same failure pattern appears twice, treat it as a signal: refine the skill instructions so the next run handles it better. Skills are not fixed artifacts. They improve.

The model will never be perfectly reliable. That is not a bug to fix. It is a condition to engineer around.

Don't get me wrong, the LLM has a lot of flexibility and freedom to do its work. It won't ask permission for every single thing it does. If it knows that there are 25 Java files that use a method called results() on class YearlySummary, and the actual method name is getResults(), when it sees that mistake it will fix it. It doesn't need me for that.

Code moves, too. An engineer finds a cleaner way to model something mid-implementation. A bug fix changes a boundary condition the spec did not anticipate. A direct code edit, by the engineer or by the LLM at the engineer's direction, improves the implementation in ways that diverge from what the spec describes. This is expected and it is not discouraged. The workflow does not treat direct code changes as violations. What it does is give divergence a name and a lifecycle. `[SPEC-DRIFT]` marks code that has moved ahead of its spec: the implementation is right, but the spec has not caught up yet. `[SPEC-INCOMPLETE]` marks the reverse: the spec is right, but the implementation is not there yet. Every marker carries a spec ID and a one-line description. The markers stay in the code until the gap is closed. Sometimes that means updating the code. Sometimes it means updating the spec to reflect what was built. The workflow runs in both directions.

---

Spec-driven development is not new. Engineers have used behavioral contracts, explicit interfaces, and traceability requirements to guide implementation for decades. This book builds on that lineage, not against it. What changed is the nature of the tool doing the translation, and more importantly, where the tool is applied. A generative model applied to code generation saves you typing. A generative model applied to the full engineering lifecycle saves you the work that was never about typing in the first place.

At some point I needed a way to refer to what I was doing. Not a formal name. Just a handle that captured the structural idea clearly enough to reason about it.

The closest term I could come up with was Spec-Surface Engineering (SSE). The surface is the spec layer: the place where intent is declared, behavior is bounded, and everything below it gets compiled from what is above it. Code sits below it. Tests do too. Diagrams, documentation, everything downstream. Human decisions happen at the surface, and the model works from what the surface specifies.

That is the pattern this book describes.

---

## Engineering the Engineering

Part I of this book lays out that discipline.

It is prescriptive. It is structured. It is opinionated. It describes the modes, the constraints, the validation gates, and the binary success conditions that make this approach work. It is, essentially, engineering the engineering. I make no apologies for this, and I am not saying this has to be done this way. I'm saying I am doing it this way and it's working, and I'm explaining why.

Part II applies that discipline to the retirement planning engine. You will see how the workflow adapts to a real system: how deterministic models are validated, how Monte Carlo simulations require existing-art-backed acceptance tests, how post-freeze validation depends on the nature of the system under development.

Part II is not a separate story. It is Part I in practice.

---

## Why This Matters Now

Most teams using LLMs for software development are using them to write code. That is a reasonable starting point. But it leaves the hardest parts of engineering untouched.

The question is not how to write code faster with LLMs.

The question is whether you are applying them to the work that actually matters.

If you are building something complex enough that requirements management, design consistency, and external validation take more time than writing code, then code generation is solving the easy problem. The hard problem, the engineering, is still on you.

It does not have to be.

The discipline described here is one way to apply LLMs to the full engineering lifecycle. It is not the only way. It is simply the way that emerged when I tried to build something I needed to trust.

If you've felt the same tension, the excitement of acceleration and the suspicion that the acceleration is aimed at the wrong target, then this book is for you.

We'll start with the discipline.

Then we'll build the system.

Note: If you're interested in reading a whitepaper that talks about this work, it's available at https://robenglander.com/writing/governing-correctness/. It's not necessary to read it in order to understand the content of this book, but if you like that kind of thing have a look.
