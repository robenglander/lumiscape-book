# Introduction
**Why Correctness Matters Now**

Large language models can produce software at a speed that would have felt absurd five years ago. Ask for a class, you get a class. Ask for a test suite, you get a test suite. Ask for a refactor, you get one. It's intoxicating.

It's also unreliable.

The problem isn't that the models are "bad." It's that they are helpful. A helpful assistant fills in gaps. It interprets ambiguity charitably. It smooths rough edges. It makes reasonable assumptions. Most of the time, that's exactly what you want.

It is exactly what you do not want in engineering.

Software that behaves according to specified intent — not probable intent, not inferred intent, not "most sensible interpretation" — requires discipline. For decades, that discipline lived in code reviews, architectural rigor, explicit contracts, and a healthy distrust of anything that "probably works."

LLMs don't remove that requirement. They amplify it.

This book is about what happens when you stop treating an LLM as a clever intern and start engineering around its behavior.

---

## The System That Forced the Discipline

The system I'm building throughout this book is a retirement planning engine. Not a demo. Not a toy project. A real engine I intend to rely on.

It models deterministic projections. It runs Monte Carlo simulations. It handles tax rules, required minimum distributions, regime switching, fat tails, portfolio sequencing, and validation across multiple interacting components. In other words, it's the kind of system where being "mostly correct" is not good enough.

I wrote it in Java. I use Maven. Those are my choices. They are not prescriptions. You could apply everything in this book in Kotlin, Go, Rust, C#, or something that hasn't been invented yet. The language is not the point. The discipline is.

This isn’t scaffolding for a methodology. The methodology showed up because the system forced it.


## How This Started

This didn't begin as a book idea.

It began because I wanted to understand my own retirement plan.

After more than four decades building and leading software systems, I found myself wanting answers I didn't fully trust from black-box tools. I wanted to see the math. I wanted to see the assumptions. I wanted to run scenarios and understand why results changed.

So I started building my own engine.

At some point, I pulled an LLM into the workflow. The productivity jump was immediate. Boilerplate disappeared. Refactors became trivial. Documentation could be generated on demand. Acceptance tests could be drafted in minutes.

And then something subtle started happening.

I would review a spec and find that a small assumption had been filled in. A validation rule implied but not stated would quietly disappear. An ambiguity would be resolved in a way that felt reasonable — but wasn't what I meant.

Nothing was obviously broken. But correctness became probabilistic.

I caught one case where a behavior I hadn't specified showed up in generated tests anyway. The tests passed. The code compiled. Everything looked fine. But the behavior was invented. It was logical. It was clean. And it was wrong.

That was the turning point.

---

## Moving the Abstraction

The breakthrough wasn't a better prompt.

It was a shift in abstraction.

Instead of letting the LLM write software directly and then reviewing the code, I moved the center of gravity upward. The spec became the authoritative source. Code became a compiled artifact. Documentation became a compiled artifact. Tests became compiled artifacts.

The spec was no longer descriptive. It was contractual.

From there, the rest followed:

- A strict spec-review workflow.
- Explicit deferral detection.
- Behavioral extraction and traceability.
- Acceptance tests derived only from declared behavior.
- Architectural metadata that controlled diagram generation.
- Multiple review passes until convergence stabilized.

And perhaps most importantly:

The model is never fully deterministic. No matter how careful the instructions.

So the workflow had to account for that.

I began insisting on repeated clean review passes before freezing anything. If the surface moved, it wasn't stable. If it wasn't stable, it wasn't ready. That simple discipline changed everything.

---

## Engineering the Engineering

Part I of this book lays out that discipline.

It is prescriptive. It is structured. It is opinionated. It describes the modes, the constraints, the validation gates, and the binary success conditions that make this approach work. It is, essentially, engineering the engineering.

Part II applies that discipline to the retirement planning engine. You will see how the workflow adapts to a real system — how deterministic models are validated, how Monte Carlo simulations require existing-art-backed acceptance tests, how post-freeze validation depends on the nature of the system under development.

Part II is not a separate story. It is Part I in practice.

---

## Why This Matters Now

Using LLMs for software development is no longer hypothetical. It has already crossed into daily reality for many engineers. Productivity gains are real. But without discipline, correctness degrades just as fast as velocity improves.

The question is no longer whether to use them.

The question is how to use them correctly.

Correct, in this book, has a specific meaning:

Software that behaves according to specified intent.

Not plausible intent.
Not inferred intent.
Specified intent.

If that sounds obvious, it's because it used to be. Code either matched the spec or it didn't. Reviews either caught the gap or they didn't. Today, the line is blurrier. Systems look correct before they are correct. They read well before they behave well.

That shift is subtle, and it's dangerous if you ignore it.

The discipline described here is one way to address it. It is not the only way. It is simply the way that emerged when I tried to build something I needed to trust.

If you've felt the same tension — the excitement of acceleration and the discomfort of uncertainty — then this book is for you.

We'll start with the discipline.

Then we'll build the system.

And along the way, we'll treat correctness as something engineered, not assumed.
