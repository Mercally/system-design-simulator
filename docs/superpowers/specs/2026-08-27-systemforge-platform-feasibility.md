# SystemForge Platform Evolution — Feasibility, Risks, Limits & Scope

Status: pre-epic analysis, approved as basis for epic breakdown. Not an implementation spec.

## 1. What was asked

Evolve SystemForge from an interview-prep simulator into a public platform architects would actually use as a no-code POC tool:

1. Extensible component library (local + cloud primitives), not hardcoded.
2. Explicit connection semantics — direct vs indirect (internal/external load balancing), pub/sub vs point-to-point vs log-replay, etc.
3. Deeper behavioral emulation — enough to catch real design mistakes, not full production fidelity.
4. Alerts for known language/framework limits (e.g. Node.js is single-core).
5. End state: the engine renders architectures live on the public site, and lets other users design an architecture, get a visual POC, and generate a prompt/code path to a working implementation via an LLM (local Ollama model, ~2GB RAM budget, or Azure AI Foundry).

Constraint locked in this session: **stay 100% client-side for the core engine**, but make persistence and behavior pluggable enough that a cloud phase 2 is a swap, not a rewrite. The one exception from day one is codegen, which needs a small backend proxy to reach the LLM (Ollama/Foundry aren't callable from a public browser directly).

## 2. Current engine, in one paragraph

`engine/simulator.ts` is a **single-pass static calculator**, not a simulator with behavior over time. One click runs one topological traversal (Kahn's algorithm) that propagates an aggregate QPS number through the graph and computes utilization/latency/bottlenecks from fixed capacity numbers (`maxQPS × replicas`). There is no clock, no event queue, no node up/down state, no memory between runs. Component *type* only affects behavior through a single hardcoded 2-entry set (`load-balancer`, `api-gateway`) checked by string equality in three places — every other component type, regardless of what it represents, fans out 100% of its traffic to every child. Scoring (`scoring/rules/*`) is similarly hardcoded to specific component IDs (`DURABLE_STORES`, entry-component lists, etc.). A basic extensibility seam already exists (`customComponentsStore` lets users add component specs at runtime), but it's data-only — 8 fields, no behavior differentiation — so a custom component today gets generic pass-through traffic and is invisible to every scoring rule.

This matters because it sets the real starting line: the "extensible library" and "connection semantics" asks are really one problem (component behavior is currently hardcoded, not data-driven), and the "deeper emulation" ask has three very different cost tiers depending on how literal you want it.

## 3. Emulation tiers (the key feasibility fork)

| Tier | What it means | Cost | Verdict |
|---|---|---|---|
| **1 — Structural semantics** | Components declare behavior as data: `fanout: broadcast \| split \| route-by-key`, `retention: bounded \| log-replayable`, `ordering`, `consumerGroups`. The engine's fan-out logic reads this instead of a hardcoded ID set. Still one click → one result. | Low–medium. Extends the existing engine; the LB special-case generalizes into the same mechanism. | **In scope for phase 1.** |
| **1.5 — Queueing theory** | Closed-form math (M/M/1, M/M/c, Erlang) computes queue length, wait time, and — critically — instability (ρ = λ/μ ≥ 1 ⇒ backlog grows unbounded) from arrival rate and service rate. No event loop needed. | Low. It's formulas, not an engine. | **In scope for phase 1.** |
| **2 — Discrete-event simulation (DES)** | A simulated clock + priority-queue of timestamped events (message arrives → dequeued → acked/retried). Each component archetype becomes a small state machine reacting to events over time — this is what actually renders "queue fills up, subscriber dies, backlog drains on recovery." | High. Well-understood 40-year-old CS (same class of algorithm as SimPy), fully feasible client-side, but it is effectively a second engine running alongside the current one. | **Explicitly deferred — phase 2+, designed for, not built now (see §6).** |

Tiers 1 and 1.5 give real, honest answers to "am I getting this wrong" (unbounded queue growth, missing redundancy, no consumer group, etc.) without the cost of Tier 2. Tier 2 is the single most expensive item in the whole vision and should be its own epic, later.

## 4. Feasibility per pillar

**Extensible component library.** Feasible now, and it's the foundation everything else sits on. The fix is turning "behavior" from hardcoded-by-ID into a small closed set of **behavior archetypes** (broadcast-fanout, point-to-point-queue, log-replayable-pubsub, capacity-passthrough, LB-split, cache-with-hit-rate, ...) that a component spec references by tag. New *component instances* using an existing archetype are fully data-only (true no-code extensibility). A genuinely new *archetype* (a new kind of behavior nobody's modeled yet) still needs a small code addition — that's an honest, worth-stating limit, not a flaw: no simulation tool gets unlimited behavioral extensibility without some code surface.

**Connection semantics (direct vs indirect).** Feasible, and it falls out of the same archetype work — "indirect via LB" becomes "downstream of a node whose archetype is LB-split," which is already how it conceptually works today, just currently hardcoded. Point-to-point vs broadcast vs log-fanout become edge/archetype-driven rather than string-matched.

**Deeper emulation (Tiers 1 + 1.5).** Feasible, scoped above. Main new work: the archetype/behavior data model, the queueing-math module, and new scoring/warning rules that read from it instead of hardcoded component IDs.

**Language/framework limitation alerts** (e.g. Node.js single-core). Feasible, same shape as existing scoring rules — a new metadata dimension on component specs (`runtimeModel: single-threaded | multi-threaded | managed-pool`, etc.) plus a rule evaluator that flags mismatches (e.g. CPU-bound workload on a single-core runtime with no clustering). Low risk, mechanical to add once the archetype/metadata pattern exists.

**Codegen via local LLM / Foundry.** Technically feasible but carries the biggest *product* risk of the five pillars, not an engineering risk: a ~2GB-RAM local model is not reliable for open-ended "write me a working app" generation. The realistic scope is **structured scaffolding from the graph** — docker-compose/IaC snippets, API stubs matching the declared connections, a README/prompt describing the architecture precisely enough for the user to hand to a *capable* coding assistant themselves. Treat "generates a fully working POC" as aspirational messaging, not a phase-1 guarantee — set expectations at "generates a strong scaffold + a precise prompt," which is honest and still valuable.

## 5. Risks

- **Small local model reliability** (codegen). Mitigate by keeping output structured/templated rather than free-form, and by treating the LLM call as one proxied backend endpoint from day one (already planned) rather than something users self-host.
- **Scope creep across 5 semi-independent pillars.** This is the reason this document exists before epics — each pillar above should become its own epic/spec, not one mega-feature.
- **Scoring rules rot if archetype migration is partial.** If component behavior moves to archetypes but scoring rules keep checking hardcoded IDs, new/custom components stay invisible to scoring even after the engine handles them correctly — the archetype migration has to touch both `engine/` and `scoring/rules/` together, not just the engine.
- **CLAUDE.md invariants must be re-validated, not assumed.** Several documented invariants (entry-node definition, exact-20-points-per-category scoring, edge data shape) are written against the *current* engine; each pillar's eventual epic needs to explicitly say which invariants survive unchanged and which are being deliberately superseded.

## 6. Designing now for Tier 2 and cloud later (the "leave it connectable" ask)

Four decisions in the phase-1 work directly determine how painful phase 2 is:

1. **Behavior as archetypes/strategies, not hardcoded IDs.** A Tier-2 DES engine would implement the *same* archetypes with time-aware event handlers instead of static formulas — the component data model doesn't change between phases, only what reads it does.
2. **Persistence behind an abstraction**, not direct `localStorage` calls scattered through the app (continues the decision already made this session) — swapping to a cloud store later is an implementation swap behind that interface.
3. **Simulation output schema left extensible/versioned** (`SimulationResult` already has this shape loosely) — so a future DES mode can emit richer output (per-tick metrics, an event log) alongside the existing instant Score tab without breaking it.
4. **Edge data keeps carrying connection semantics as data** (protocol/archetype-derived routing), so DES can reuse the same edges instead of a parallel graph model.

None of this requires building Tier 2 or the cloud backend now — it requires not making phase-1 decisions that would have to be undone to add them later.

## 7. Explicitly out of scope for this phase

- Real-time animated simulation (Tier 2 DES).
- Any backend beyond the single LLM-proxy endpoint for codegen.
- Full functional code generation ("push button, get a deployed app").
- Multi-user/shared designs, accounts, cloud persistence.

## 8. Suggested epic breakdown (for the next step, not decided here)

1. Component behavior archetypes (engine + scoring migration off hardcoded IDs) — foundation, unblocks everything else.
2. Connection semantics on edges (direct/indirect, pub/sub/point-to-point/log) — builds on #1.
3. Tier-1.5 queueing math + new design-error warnings.
4. Language/framework limitation rules.
5. Codegen scaffolding (LLM proxy endpoint + prompt/template generation from the graph).
6. (Later, separate track) Tier-2 DES engine + live playback.
7. (Later, separate track) Cloud persistence/accounts.
