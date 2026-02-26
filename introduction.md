# Introduction
**Why Correctness Matters Now**

Large language models can write code at a speed that would have felt absurd five years ago. Ask for a class, you get a class. Ask for a test suite, you get a test suite. Ask for a refactor, you get one. It's intoxicating.

It's also unreliable.

The problem isn't that the models are "bad." It's that they are helpful. A helpful assistant fills in gaps. It interprets ambiguity charitably. It smooths rough edges. It makes reasonable assumptions. Most of the time, that's exactly what you want.

It is exactly what you do not want in engineering.

Software that behaves according to specified intent — not probable intent, not inferred intent, not "most sensible interpretation" — requires discipline. For decades, that discipline lived in code reviews, architectural rigor, explicit contracts, and a healthy distrust of anything that "probably works."

LLMs don't remove that requirement. They amplify it.

This book is about what happens when you stop treating an LLM as a clever intern and start engineering around its behavior.

---

## What This Book Is Not

If you are fixing a bug in an existing system, or asking an LLM to write a utility method, or adding a small feature to a codebase you already own, go ahead. You do not need this. The LLM will be useful and you will be fine. This book is not about that.

This book is about what happens when the project has a substantial perimeter. A system large enough where cognitive overload is inevitable. A domain with correctness requirements that cannot be fudged. A simulation with interacting components where "looks right" is not good enough. When the perimeter is small, engineering discipline might be overhead. When the perimeter is large, it is the only thing that keeps the system coherent.

If you are in the middle of a large project and realize it lacks this kind of structure, that is also relevant. But the starting point is recognizing whether the perimeter warrants it. This book assumes it does.

This is also not an attempt to invent a new engineering methodology. The practices here — behavioral contracts, explicit specifications, acceptance tests, traceability matrices, test-driven development — have a long history. Engineers have done this for decades. What is different is the context: implementing that discipline when the entity translating spec to artifact is a generative model rather than a deterministic compiler or a human following documented procedures. The discipline is established. The adaptation is new.

---

## The System That Forced the Discipline

The system I'm building throughout this book is part of a retirement planning engine. Not a demo. Not a toy project. A real engine I intend to rely on.

It models deterministic projections. It runs Monte Carlo simulations. It handles tax rules, required minimum distributions, regime switching, fat tails, portfolio sequencing, and validation across multiple interacting components. In other words, it's the kind of system where being "mostly correct" is not good enough.

I wrote it in Java. I use Maven. Those are my choices. They are not prescriptions. You could apply everything in this book in Kotlin, Go, Rust, C#, etc. The language is not the point. The discipline is.

This isn't scaffolding for a methodology. The methodology showed up because the system forced it.


## How This Started

This didn't begin as a book idea.

It began because I wanted to understand my own retirement plan.

After more than four decades building and leading software systems, I found myself wanting answers about how black-box tools were deriving results. I wanted to see the math. I wanted to see the assumptions. I wanted to run scenarios and understand why results changed.

So I started building my own engine.

At some point, I pulled an LLM into the workflow. The productivity jump was immediate. Refactors became trivial. Documentation could be generated on demand. Acceptance tests could be drafted in minutes.

And then something subtle started happening.

I would review a spec and find that a small assumption had been filled in. A validation rule implied but not stated would quietly disappear. An ambiguity would be resolved in a way that felt reasonable — but wasn't what I meant.

Nothing was obviously broken. But correctness became probabilistic.

A gap in the spec is not neutral. It is a license for the model to fill intent with plausible inference. The inferences are frequently wrong in ways that are invisible until they have been compiled into artifacts and validated against tests derived from the same inference.

I caught one case where a behavior I hadn't specified showed up in generated tests anyway. The tests passed. The code compiled. Everything looked fine. But the behavior was invented. It was logical. It was clean. And it was wrong.

That was the turning point.

---

## Moving the Abstraction

The breakthrough wasn't a better prompt.

It was a shift in abstraction.

Instead of letting the LLM write software directly and then reviewing the code, I moved the center of gravity upward. The spec became the authoritative source. Everything downstream from it became compiled artifacts: code, tests, documentation, diagrams.

The spec was no longer descriptive. It was contractual.

From there, the rest followed:

- A strict spec-review workflow.
- Explicit deferral detection.
- Behavioral extraction and traceability.
- Acceptance tests derived only from declared behavior.
- Architectural metadata that controlled diagram generation.
- Multiple review passes until convergence stabilized.
- Machine-readable manifests that record what each mode checked and what evidence supports each conclusion. A vague entry like "evidence: checked" is an audit trail violation. Evidence must be specific.

And perhaps most importantly:

The model is never fully deterministic. No matter how careful the instructions.

So the workflow had to account for that.

One specific problem needed a dedicated rule. The model has no persistent cross-session state. A decision made in yesterday's session does not exist in today's context window. An architectural constraint established two weeks ago is invisible unless it is re-stated or embedded in a file the model reads right now. Left unaddressed, this means the model can assert things about the spec surface based on vague recollection rather than current file contents. The fix is structural. Every behavioral claim must be grounded in a tool call from the current session. Not memory. Not inference from prior conversations. A concrete tool call: grep the file, read the section, show the evidence. This is the NO-MEMORY RULE, and it applies to every skill in the pipeline without exception.

I began insisting on repeated clean review passes before freezing anything. If the surface moved, it wasn't stable. If it wasn't stable, it wasn't ready. That simple discipline changed everything.

---

## The Engineer Remains the Engineer

The abstraction shift does not change who is responsible.

On my retirement planning engine project I am the architect. I define the module boundaries, the behavioral contracts, the dependency structure, and the acceptance criteria. The model generates artifacts — specs, tests, code, diagrams — based on those contracts. It does not decide what to build. It builds what is specified.  And most importantly if the outcome is wrong that's my fault. There's no tolerance for abdicating responsibility.

This distinction matters in practice. Nothing in this workflow executes without explicit authorization. The model proposes an action, explains what it will do and why, and waits. I review it. I approve it or I don't. There is no sweeping approval. There is no "run everything." Each action is deliberate.

That deliberation happens in conversation. You invoke a mode, Claude runs it, reports its findings, and the exchange continues from there: findings, approvals, directions, decisions. The execution medium is unchanged from the ad-hoc LLM usage you already know. What changed is the grounding. The conversation operates against a spec surface you control, with constraints loaded from files, not stated in prompts and forgotten.

None of this makes the model reliable. It will still make mistakes. It will skip a step in a skill and not mention it. It will assert something about a spec it has not actually read in this session. It will produce a plausible-looking output that missed the point. The structure catches many of these: the evidence requirement in the manifest, the freeze gate, the no-memory rule. But it does not catch all of them.

That is still your job. If something looks off, stop it. Question the evidence. Ask it to re-read the section. When a result seems too clean or arrives too fast, investigate. And when the same failure pattern appears twice, treat it as a signal: refine the skill instructions so the next run handles it better. Skills are not fixed artifacts. They improve.

The model will never be perfectly reliable. That is not a bug to fix. It is a condition to engineer around.

The model does not fill gaps. That is not a limitation I work around — it is a constraint I enforce. When the model encounters an ambiguity, the correct response is to stop and report it, not resolve it charitably. When a spec is incomplete, the correct response is to flag the gap, not invent reasonable behavior. The model proposes. The engineer decides. Accountability cannot be delegated. If the system can act without me, I am no longer engineering it.  DOn't get me wrong, the LLM has a lot of flexibility and freedom to do its work.  It won't ask permission for ever single thing it does.  For example, if it knows that there are 25 Java files that use a method called results() on class YearlySummary, and the actual method name is getResults(), when it sees that mistake it will fix it.  It doesn't need me for that.

Code moves, too. An engineer finds a cleaner way to model something mid-implementation. A bug fix changes a boundary condition the spec did not anticipate. A direct code edit, by the engineer or by the LLM at the engineer's direction, improves the implementation in ways that diverge from what the spec describes. This is expected and it is not discouraged. The workflow does not treat direct code changes as violations. What it does is give divergence a name and a lifecycle. `[SPEC-DRIFT]` marks code that has moved ahead of its spec: the implementation is right, but the spec has not caught up yet. `[SPEC-INCOMPLETE]` marks the reverse: the spec is right, but the implementation is not there yet. Every marker carries a spec ID and a one-line description. The markers stay in the code until the gap is closed. Sometimes that means updating the code. Sometimes it means updating the spec to reflect what was built. The workflow runs in both directions.

---

Spec-driven development is not new. Engineers have used behavioral contracts, explicit interfaces, and traceability requirements to guide implementation for decades. This book builds on that lineage, not against it. What changed is the nature of the tool doing the translation. A deterministic compiler takes a specification and produces a predictable artifact: same input, same output, every time. A generative system takes a specification and produces a plausible artifact, shaped by training, context, and probability. Keeping that artifact aligned with the surface it was compiled from requires a discipline that deterministic tools never needed.

At some point I needed a way to refer to what I was doing. Not a formal name. Just a handle that captured the structural idea clearly enough to reason about it.

The closest term I could come up with was Spec-Surface Engineering (SSE). The surface is the spec layer — the place where intent is declared, behavior is bounded, and everything below it gets compiled from what is above it. Code sits below it. Tests do too. Diagrams, documentation, everything downstream. Human decisions happen at the surface, and the model works from what the surface specifies.

That is the pattern this book describes.

---

## Engineering the Engineering

Part I of this book lays out that discipline.

It is prescriptive. It is structured. It is opinionated. It describes the modes, the constraints, the validation gates, and the binary success conditions that make this approach work. It is, essentially, engineering the engineering.  I make no apologies for this, and I am not saying this has to be done this way.  I'm saying I am doing it this way and it's working, and I'm explaining why.

Part II applies that discipline to the retirement planning engine. You will see how the workflow adapts to a real system — how deterministic models are validated, how Monte Carlo simulations require existing-art-backed acceptance tests, how post-freeze validation depends on the nature of the system under development.

Part II is not a separate story. It is Part I in practice.

---

## Why This Matters Now

Using LLMs for software development is no longer hypothetical. It has already crossed into daily reality for many engineers. Productivity gains can be real. But without discipline, correctness degrades just as fast as velocity appears to improve.

The question is no longer whether to use LLMs.

The question is how to use them correctly.

Correct, in this book, has a specific meaning:

Software that behaves according to specified intent.

Not plausible intent.
Not inferred intent.
Specified intent.

If that sounds obvious, it's because it used to be. Code either matched the spec or it didn't. Reviews either caught the gap or they didn't. Today, the line is blurrier. Systems look correct before they are correct. They read well before they behave well.

That is silent divergence. The system passes its tests, ships its features, and quietly becomes less aligned with what it was meant to do. It does not announce itself. It accumulates.

That shift is subtle, and it's dangerous if you ignore it.

The discipline described here is one way to address it. It is not the only way. It is simply the way that emerged when I tried to build something I needed to trust.

If you've felt the same tension — the excitement of acceleration and the discomfort of uncertainty — then this book is for you.

We'll start with the discipline.

Then we'll build the system.

And along the way, we'll treat correctness as something engineered, not assumed.

Correctness is not a property of models. It is a property of discipline.

Note:  If you're interested in reading a whitepaper that talks about this work it's available at https://robenglander.com/writing/governing-correctness/.  It's not necessary to read it in order to understand the content of this book, but if you like that kind of thing have a look.
