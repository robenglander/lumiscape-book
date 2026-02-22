# Governing Correctness in LLM-Assisted Development
## Probabilistic generation changes where software fails — and what engineers must govern in response

**Robert Englander**

---

A team builds a compliance service from a reviewed and approved specification. The service compiles. Tests pass. The system ships. Six weeks later, an audit finds that the service is applying a validation threshold that was removed from the specification before implementation began. The threshold appeared in an earlier draft. That draft was used as generation context. The final specification was not consulted directly. No test caught it because the tests were generated from the same context as the implementation. Every artifact was internally consistent. The system was wrong.

This is not a generation quality problem. It is a boundary problem. The failure did not originate in the model. It originated at the transition between the specification and the generated artifacts — a boundary the team had not explicitly governed. LLMs don't break software correctness — they relocate where correctness must be governed.

---

**This article argues:**
1. Artifact validity does not guarantee correctness. A generated artifact can be syntactically correct, pass its test suite, and still fail to realize specified intent.
2. Interpretive boundaries — the transitions between specification, generated artifacts, and runtime behavior — introduce probabilistic misalignment that single-layer techniques such as prompt engineering and evaluation harnesses cannot prevent.
3. Engineering must shift from validating artifacts against each other to aligning artifacts against specifications at each boundary crossing.

Throughout this article, correctness means system behavior that matches specified intent — not reliability, not statistical accuracy, not formal verification.

---

## The Mechanism: Drift at Boundaries

LLMs perform probabilistic translation: given structured or unstructured intent, they generate artifacts that are statistically aligned with the prompt. The failure mode this introduces is not hallucination in the dramatic sense. It is drift — subtle semantic shifts, implicit assumption filling, interpretation under ambiguity, compression of meaning at transitions. Research on semantic drift in generative systems confirms this: models produce plausible continuations that diverge from factual grounding without signaling that divergence at any individual step (Meta AI Research 2024). Generative models optimize for local likelihood, not global semantic consistency (Cheung et al. 2025) — which means generation quality at any individual step does not prevent misalignment at the system level.

Drift does not announce itself. It accumulates. A system can pass its test suite, compile cleanly, and ship features while becoming progressively misaligned with its specified intent. The misalignment is invisible at every layer — each step looks locally reasonable — and only visible when artifacts are compared against the specification.

Current engineering practice addresses this inside a single layer. Prompt engineering reduces ambiguity in one translation step. Structured output constraints shape the form of generation. Evaluation harnesses detect observable failures. None of them govern the relationship between specification, artifacts, and runtime behavior across boundaries. Correctness can degrade at each crossing even when every individual step looks fine.

This explains a pattern that many teams experience but struggle to name. Systems appear stable at the prompt level. They produce valid structured outputs. They pass evaluation suites. And yet, over time, behavior diverges from the original intent. Nothing breaks. Correctness erodes.

There is a structural reason for this. The engineer holds system coherence across time, sessions, and modules. A constraint established two sessions ago, an invariant that spans ten specifications, a vocabulary decision made last week — all of these live in the engineer's understanding of the system. The model operates within its context window. Any constraint outside that window will not be maintained. This is not a model failure. It is a structural property of the tool. Every constraint the engineer cares about must be externalized into the specification — not held in memory or assumed to persist across sessions.

Drift is the natural outcome of probabilistic transformation across ungoverned boundaries. The solution is not a better model. It is explicit boundary governance.

---

## A Layered Model of Correctness

Correctness is defined in the specification and must survive through translation, realization, and observation. It does not propagate automatically. Each interpretive boundary is a place where it can degrade, silently, without signaling failure to any individual layer.

The system spans five planes. At the top is the intent plane — specifications, domain rules, invariants, constraints — where correctness is defined. Below it lies the primary translation plane, where probabilistic generation converts structured intent into artifacts. Refinement is a distinct plane: iterative correction introduces its own drift risk, separate from initial generation. The artifact plane is the observable system — code, tests, documentation. At the bottom, the observation plane applies measurement, validation, and empirical checks.

Five interpretive boundaries separate these planes. The interpretation boundary lies between intent and primary translation: where specification becomes a form the model can act on, and where any ambiguity compressed here propagates forward into every downstream plane. The alignment boundary falls between translation and artifacts: does the generated code faithfully reflect the specification? The realization boundary sits between artifacts and runtime: where code becomes executing behavior. The validation boundary operates within the observation plane: where runtime behavior is measured against expectations. The reconciliation boundary closes the loop: where deviations feed back into refinement through another generation step, with its own drift potential.

Correctness degrades at interpretive boundaries. Governing them is the engineering task. The value of naming both planes and boundaries is diagnostic: when correctness fails, the framework identifies where it failed and why the failure was invisible at adjacent layers.

---

## Where Correctness Fails

**At the interpretation boundary**, correctness is threatened before generation begins. When intent becomes structured specification, meaning is compressed. Ambiguities surface. Assumptions leak. In deterministic systems this boundary is often implicit — compiler and author share a formal language that constrains compression. In generative systems it must be made explicit, because the downstream translator will fill every gap without signaling that it has done so. An ambiguity in the specification will be resolved at translation time, and the resolution will look authoritative to every subsequent validation pass.

Gaps in the intent plane are not neutral. They are licenses for the model to fill intent with plausible inference. Those inferences will frequently be wrong in ways that are invisible until realization — by which point the gap has been compiled into artifacts, validated against tests derived from the same inference, and shipped.

The interpretation boundary also has a cross-specification dimension. A specification can be internally consistent and still be wrong in the context of the system it belongs to. When a normative source is updated — a vocabulary extended, a field removed, an interface changed — dependent specifications that were not updated carry stale intent.

A concrete instance: a normative specification defines the valid action targets for a routing component. A subsequent revision expands the list, replacing one grouped target with four fine-grained ones. The normative source is updated. The cascade to ten dependent specifications is incomplete. Each passes individual review — internally consistent, structurally complete. Together, they describe a system where the component that produces outputs and the component that consumes them disagree on what values are valid. No individual specification is wrong. The system as a whole cannot be implemented consistently.

**At the alignment boundary**, the question is faithfulness. Do the artifacts reflect the specified intent? Not elegance, not efficiency — faithfulness. The model cannot self-report misalignment. An artifact that filled a gap with plausible inference will look correct, compile, and may pass tests written against the same inference rather than the specification. The misalignment is invisible unless the artifact is compared directly to the specification. The compliance service failure from the opening is an alignment boundary failure: the authoritative specification existed but was not the reference for generation. The stale draft was. Every artifact was consistent with the draft. None were consistent with the final specification.

Governing this boundary also requires that any conflict encountered during generation be treated as a formal decision point, not a resolution opportunity. When a specification references a type not defined anywhere, or two specifications make contradictory claims about the same interface, the correct response is to stop, document the conflict, and wait for an explicit direction. Resolution by assumption is not resolution. It is correctness erosion, formalized.

**Illustrative case.** A service specification defines a spending floor as an inflation-adjusted value. Code generation produces an implementation that computes the floor against the nominal base-year value — a locally reasonable interpretation of an underspecified term. Tests are generated from the implementation and pass. In runtime conditions where inflation is significant, the floor is wrong. The misalignment is only detectable by comparing implementation semantics to specification text, not by checking artifacts against each other.

**At the realization boundary**, artifacts become runtime behavior. Two independently generated components can each be internally correct and still fail when they meet at runtime — not because either is wrong in isolation, but because they were generated against specifications that did not fully resolve their shared interface. This is a distinct failure from alignment boundary failures: each component traces to its specification, each passes its tests, and the incompatibility only surfaces in execution. In one production system, an integration check found that the deterministic engine compared terminal portfolio value against a nominal spending floor while the stochastic engine compared against an inflation-adjusted floor. Neither was internally wrong — each traced cleanly to its specification. Together they produced results that could not be compared. The fix required an explicit semantic decision about what "floor" meant across both engines, logged as a formal revision. Correctness must survive realization. Generation success does not guarantee it.

**At the validation boundary**, the threat is internal anchoring. Without empirical grounding, validation measures consistency — does the artifact match the specification? — not correctness — does the specification correctly reflect external reality? A specification can be internally consistent and externally wrong. In one production system, empirical validation caught a specification that had pinned an outdated regulatory table for a mandatory financial calculation. The specification was structurally complete and had passed every review pass. The empirical check — comparing expected outputs against the current published authority — found discrepancies at multiple input values. Without that external anchor, the error would have been compiled into every calculation before it surfaced. The specification was the authority. The authority was wrong.

**At the reconciliation boundary**, correction introduces its own misalignment risk. Each refinement pass works from the output of the prior pass, not from the original specification. Corrections can compound as readily as they resolve. An artifact can be refined into internal consistency — all parts agreeing with each other — while drifting from the specification that produced it. The artifact looks correct. The tests pass. The misalignment has been laundered into the artifact's internal coherence. The artifact is now self-consistent in its incorrectness — and every subsequent validation pass confirms the consistency, not the correctness.

---

## Governing the Interpretive Boundaries

Specification-driven development and formal methods established the core discipline here decades ago: structured intent should precede implementation, and artifacts should be traceable to requirements. What is new is the application to a tool that introduces probabilistic variability at every boundary. A deterministic compiler's behavior at the alignment boundary is fully specified by its grammar: same input, same output, every time. A generative system's behavior at the same boundary is probabilistic, context-dependent, and variable across invocations.

Boundary governance must scale with boundary volatility. We need a term for this class of practice. I'll refer to it as Spec-Surface Engineering (SSE).

Spec-Surface Engineering is not a new development process, framework, or methodology. It treats the specification as the governing interface between intent and generated artifacts — where correctness must be enforced.

In SSE, the specification is the governing surface. Everything compiled from it — code, tests, documentation — derives its correctness authority from it. The engineer works at the surface. The model works from it. This has a consequence beyond code correctness: it eliminates documentation drift. In conventional development, documentation and code have different sources and can diverge. When both are compiled from the same specification, that structural cause is removed. Misalignment between documentation and runtime behavior is not managed. It is architecturally prevented.

SSE implements governance as a sequential, artifact-gated pipeline. Each gate ensures no phase begins until the preceding interpretive boundary has been explicitly closed.

**Closing the interpretation boundary.** Every specification undergoes systematic review before any generation begins. Review enforces named behavioral contracts — independently testable statements of intent — and prohibits deferral language, vague validation rules, and implicit assumptions. A validation rule that says "inputs are validated" without specifying rejection conditions, error types, and the exact invalid inputs is an open boundary — one the model will close in its own way. A specification cannot proceed until it passes. The gate is binary.

A second pass operates across the entire specification surface. A specification can pass individual review and still be wrong in context. This pass detects dependent specification drift, stale vocabulary, and conflicting assertions across specifications. The methodology is evidence-first: surface raw results before drawing conclusions. No consistency assertion is accepted without showing the data. Terms changed in a normative source must cascade to every specification that depends on them. A vocabulary extended in one specification and not updated in its dependents is a correctness failure waiting to be compiled.

**Boundary commitment.** The specification freeze is the moment of explicit commitment. Once frozen, the specification is immutable except through a formal revision request: a structured protocol that names the conflict by type — a missing definition, a cross-specification contradiction, a module boundary violation — presents options with their tradeoffs, and waits for an explicit decision before any revision proceeds. Every post-freeze revision is logged. In one production system, eight revisions were required after the initial freeze — each triggered by a subsequent validation pass detecting misalignment that specification review had not caught.

**Empirical validation.** Before generation begins, the specification is validated against external authoritative sources: regulatory publications, actuarial standards, canonical mathematical properties for statistical components. A specification that is internally consistent but factually wrong fails this gate. A regulatory table pinned to the wrong year, a statistical property that contradicts established literature, a calculation that produces wrong results on known inputs — all fail. Subjective confidence is not a passing criterion.

**Governed generation.** Generation proceeds under a discipline: every conflict encountered triggers a formal revision request. The model does not resolve ambiguity by assumption, infer missing definitions, or reorganize components without explicit direction. A traceability matrix — behavioral contract to acceptance test to generated code to unit test — makes gaps visible. A behavior specified but not tested is a gap. A test written from the implementation rather than the specification is a gap, even if it passes. A missing link is evidence of incomplete correctness.

**Post-generation alignment check.** This is the gate most teams skip. It is a deliberate comparison of the completed artifacts against the frozen specification: does every declared behavioral contract have a corresponding implementation? Does every implementation behavior trace to a declared contract? Does the implementation semantics of each behavior match the declared intent — not just structurally, but in what the code does?

This last question is the hardest. A behavioral contract can appear in both the specification and the code, the traceability matrix can record a link, and the implementation can still do something different from what the specification meant. In the same production system, the first alignment check found this for the spending floor case described earlier — the test linked to the behavioral contract was passing because it had been written from the implementation, not from the specification. The link existed. The test was green. The runtime behavior was wrong.

Until the alignment check is complete, the alignment boundary is not closed. It is merely uncontradicted.

---

## Validation and Reconciliation

These are distinct operations, and conflating them is a source of structural failure.

Validation answers: does observable behavior align with expectations? It is a measurement. It produces a result per criterion. It does not change the artifact. Reconciliation answers: what change is required to restore correctness? It is a directed intervention — and it is itself a probabilistic translation step, with its own potential to introduce drift.

Reconciliation without validation introduces new misalignment: the intervention is made without a measurement baseline, and the correction may compromise correctness elsewhere while resolving the immediate problem. Validation without reconciliation documents misalignment without addressing it.

Both must be explicit and repeatable. An undocumented reconciliation is indistinguishable from a new translation: future validation cannot determine whether the current state reflects an intentional correction or an untracked drift event. In a governed system, every post-freeze reconciliation is documented — the conflict identified, the options considered, the decision made, the specifications affected. That record is not overhead. It is the audit backbone that makes the system's correctness history legible. When a decision made six months ago produces unexpected behavior today, there is an answer. Without that record, correctness history becomes reconstruction. Reconstruction is itself an interpretive act. Interpretive acts introduce drift.

---

## What Must Change

The failure mode described throughout this article — correctness that looks intact at every layer but has silently diverged from specification — does not require unusual circumstances. It requires only ungoverned boundary crossings and the default assumption that artifact validity implies specification alignment. Without an active engineer governing the spec surface, probabilistic generation will optimize for plausibility, not correctness.

Three assumptions must change for engineers working with generative tools:

**The reference for validation must be the specification, not other artifacts.** Testing generated code against generated tests confirms consistency between two probabilistic outputs from the same generation context. It does not confirm alignment with the specification. Every validation check must trace back to a specification statement.

**Boundary crossings must be explicit, not assumed.** The transitions between specification, generated artifacts, and runtime behavior are not implementation details. They are correctness risks. Each one requires an explicit governance mechanism — not a process step that assumes the crossing happens correctly, but a gate that verifies it.

**Correctness must be governed before generation begins, not recovered after.** An error introduced at the interpretation boundary compiles into artifacts, generates consistent tests, and produces internally coherent runtime behavior. By the time it is observable, it has been laundered through every subsequent boundary crossing. The correction instinct — catch it in testing, fix it in revision — does not scale to misalignment that has propagated across the full boundary stack. Governing boundaries before generation is not process overhead. It is the only point in the lifecycle where correction is straightforward.

Generative tools increase productivity at the cost of increased boundary volatility. The response is not to reduce tool use. Governing means making each boundary crossing explicit — a gate that verifies, not a process step that assumes. It means treating the specification as authority, not documentation, and owning every reconciliation decision before the next generation step begins.

---

## References

Cheung, A., et al. "LLM-Based Code Translation Needs Formal Compositional Reasoning." Technical Report No. EECS-2025-174. Berkeley: University of California, 2025. https://www2.eecs.berkeley.edu/Pubs/TechRpts/2025/EECS-2025-174.pdf

Meta AI Research. "Know When To Stop: A Study of Semantic Drift in Text Generation." Meta AI. 2024. https://ai.meta.com/research/publications/know-when-to-stop-a-study-of-semantic-drift-in-text-generation/

---

*Robert Englander spent more than four decades building and leading software systems, most recently as Lead Principal Engineer at Atlassian. He has retired from corporate life and now works as an independent researcher, advisor, and consultant focused on large-scale distributed systems, desktop architecture, and software engineering in the age of AI. He is the author of several O'Reilly books and has spoken at numerous industry conferences. This article was developed with the assistance of Claude (Anthropic) — the author governed the process, the model executed within those constraints — the discipline this article describes, applied to the article itself.*
