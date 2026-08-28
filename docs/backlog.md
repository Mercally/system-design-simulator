# SystemForge Platform Evolution — Backlog

Status: **draft, not pushed to GitHub yet.** Source: `docs/superpowers/specs/2026-08-27-systemforge-platform-feasibility.md` (approved feasibility analysis). Holding here until told to convert into GitHub issues — suggested conversion: one issue per epic, its "Scope" bullets become the issue's task checklist.

Priority key: **P0** foundation (blocks everything else) · **P1** core value of phase 1 · **P2** phase-1 stretch · **P3** explicitly deferred to phase 2, tracked but not started.

---

## EPIC-00 — Showcase view (highest priority, independent track)

Full design: `docs/superpowers/specs/2026-08-27-showcase-view-design.md`. Not part of the platform-evolution track below — ships real deployed architectures as a portfolio/embed feature, has no dependency on EPIC-01..07 and nothing downstream depends on it.

**Goal.** A generic, reusable showcase system for presenting architectures the user has actually deployed. First entry: "DTE Masters" (real multi-tenant SaaS). Gallery view (`/showcase`) lists all showcased architectures; a standalone read-only view per architecture (`/showcase/[slug]`) is embeddable via iframe into `mercally.com` (separate Angular site, same domain family).

**Depends on:** nothing.

**In scope**
- `/showcase` gallery route + `/showcase/[slug]` embeddable read-only route (new App Router routes; today the app has only `/`).
- Static JSON per architecture (`data/showcase/*.json`), authored by Claude for this pass — not drawn by the user yet.
- New component pool (`data/showcaseComponents.ts`), separate from the interview-prep library (`data/components.ts`): 4 generic archetypes with per-instance label override (`payment-gateway`, `background-job-processor`, `reverse-proxy`, `compute-host`) + 8 cloud-provider-specific specs (`azure-blob-storage`, `azure-key-vault`, `azure-app-service`, `azure-app-registration`, `microsoft-graph`, `gcp-artifact-registry`, `gcp-pubsub`, `gmail-api`).
- One small core-engine addition: optional per-instance `description` field on `ComponentNodeData` + Props panel input — a real editor capability any design can use, reused by the showcase click-panel, not a showcase-only fork.
- `next.config.ts` CSP `frame-ancestors` header scoped to `/showcase/:path*` only, allow-listing the mercally.com domain (placeholder until real hosting/domain is decided — not deployed yet).

**Out of scope:** hosting/deployment itself, responsive iframe auto-height (v1 uses fixed height), any interview-mode UI (Score/Sim/Capacity/Tradeoffs) in the showcase view, a visual authoring UI for showcase JSON.

**Size:** M. New routes + new data pool + one small core schema addition; DTE Masters content authoring is the most time-consuming single piece.

**Tasks**
1. Add `description?: string` to `ComponentNodeData` (`store/canvasStore.ts`) + wire through node creation/update paths.
2. Add the description input to the node Props tab (`components/panel/RightPanel.tsx`).
3. Create `data/showcaseComponents.ts` with the 12 specs (4 generic archetypes + 8 cloud-specific).
4. Build the showcase component-lookup helper (checks `data/components.ts` then `data/showcaseComponents.ts`), mirroring the existing `customComponentsStore` fallback pattern.
5. Author `data/showcase/dte-masters.json` — model DTE Masters' real infra as nodes/edges, captioned via `label`/`description`.
6. Create `data/showcase/index.ts` listing showcase entries.
7. Build `/showcase` gallery route (card grid, visual pattern borrowed from `ProblemSelector`).
8. Build `/showcase/[slug]` route: minimal shell, canvas-only `readOnly` render (reuse the existing reference-solution readOnly mechanism), node-click panel showing `label`/`description`.
9. Add the `next.config.ts` `frame-ancestors` header rule scoped to `/showcase/:path*` (placeholder domain, flagged to update at deploy time).
10. Manual QA: `/showcase/dte-masters` standalone (no editor chrome leaks) + embedded in a local test `<iframe>` before calling this done.

**Done when:** `/showcase` lists DTE Masters, `/showcase/dte-masters` renders correctly standalone and inside a test iframe with no editor chrome, and adding a second architecture requires zero route/code changes (data + component specs only).

---

## P0 — EPIC-01: Component behavior archetypes

**Goal.** Replace the hardcoded `Set(["load-balancer","api-gateway"])` fan-out check (`engine/simulator.ts`) and the hardcoded component-ID lists in `scoring/rules/*` (`DURABLE_STORES`, entry-component lists, etc.) with a small closed set of data-driven **behavior archetypes** (e.g. `broadcast-fanout`, `point-to-point-split`, `log-replayable-pubsub`, `capacity-passthrough`) that any component spec — built-in or custom — references by tag.

**Depends on:** nothing. This unblocks EPIC-02, 03, 04.

**In scope**
- Archetype field(s) added to `SystemComponent` schema.
- Simulator fan-out logic reads archetype instead of ID matching.
- Scoring rules migrated to read archetype/metadata instead of hardcoded IDs, so custom components become visible to scoring for the first time.
- Existing 30 built-in components tagged with the correct archetype (no behavior change for existing designs — this is a refactor, not a feature, from the user's point of view).

**Out of scope:** letting users define an entirely new archetype without code (see feasibility doc §4 — that's an honest, permanent limit, not deferred work).

**Why this priority.** Every other pillar (connections, queueing math, framework-limit rules, even codegen's ability to describe a design precisely) depends on component behavior being data-driven instead of string-matched. Nothing else in the backlog can start cleanly before this lands.

**Size:** M. Touches `engine/simulator.ts`, all of `scoring/rules/`, `data/components.ts`, `types/component.ts`. Existing invariant "each category rule totals exactly 20" must survive — re-verify arithmetic per CLAUDE.md.

**Tasks**
1. Define the archetype taxonomy (name + exact semantics for each) — keep the set small; this is a design decision, not a code task.
2. Add `archetype` field to `SystemComponent`/`ComponentSpec` (`types/component.ts`).
3. Tag all 30 built-in components in `data/components.ts` with the correct archetype.
4. Update `customComponentsStore.ts` so custom components pick an archetype from the closed set (no free text).
5. Replace the `LOAD_BALANCING_COMPONENTS` Set check in `engine/simulator.ts` (the 3 call sites) with archetype-based dispatch.
6. Regression: re-run all 35 reference-solution problems, confirm simulator output (throughput/utilization/bottlenecks) is byte-for-byte unchanged.
7. Migrate `scoring/rules/availability.ts` (`DURABLE_STORES`, `entryComponents`) off hardcoded ID lists to archetype/metadata checks.
8. Migrate the remaining rule files (`scalability.ts`, `latency.ts`, `cost.ts`, `tradeoffs.ts`) the same way.
9. Regression: re-run `scorer.ts` against all 35 reference solutions, confirm each category's score is unchanged.
10. Update CLAUDE.md's "Key invariants" section to describe archetype-based behavior instead of the old hardcoded Set.

**Done when:** a custom component using an existing archetype gets correct fan-out behavior *and* shows up correctly in all 5 scoring categories, with zero change to scores of existing reference solutions (regression check).

---

## P1 — EPIC-02: Connection semantics on edges

**Goal.** Make direct-vs-indirect and pub/sub-vs-point-to-point-vs-log connection semantics explicit and inspectable, instead of implicit in "is the source node's ID in a hardcoded set."

**Depends on:** EPIC-01 (archetypes are what an edge's behavior derives from).

**In scope**
- Edge data model gains the fields needed to express: direct connection, indirect-via-LB (internal/external), broadcast/topic fanout, point-to-point routing.
- UI affordance so a user can see/understand which kind of connection they've drawn (not necessarily new visual design — functional clarity first).
- `conceptLibrary.ts`/interview-facing copy updated if any existing explanation of "how load balancing works in this tool" becomes inaccurate.

**Out of scope:** animating traffic differently per connection type (that's Tier 2 / EPIC-06 territory).

**Size:** S–M, mostly data-model + store (`canvasStore.ts` edge shape) once EPIC-01 exists.

**Tasks**
1. Design the connection-type taxonomy (direct, indirect-via-internal-LB, indirect-via-external-LB, broadcast/topic, point-to-point-routed) and decide which are inferred from the source node's archetype vs. explicitly set on the edge.
2. Extend `CustomEdgeData` (`canvasStore.ts`) with the new field(s).
3. Update the connect-time edge default (`canvasStore.ts` ~line 325) to set a sensible default from the source node's archetype.
4. Surface connection type visibly on canvas (label/icon/edge style) — functional clarity first, not a redesign.
5. Add connection-type view/override to the edge Props tab in `RightPanel`.
6. Update any `conceptLibrary.ts` entries whose load-balancing/connection explanation becomes inaccurate.
7. Ensure `SerializedEdge`/the export-import envelope carries the new field (CLAUDE.md: edge `data` must round-trip on save/load).
8. Regression: confirm designs saved before this change (no connection-type field) load with a sane inferred default.

---

## P1 — EPIC-03: Queueing-theory (Tier 1.5) design-error checks

**Goal.** Add closed-form queueing math (M/M/1, M/M/c, Erlang) so the tool can flag real instability (ρ = λ/μ ≥ 1 → unbounded backlog), not just utilization percentage. This is the concrete answer to "help me know if I'm getting the design wrong," without building a time-stepped engine.

**Depends on:** EPIC-01 (needs archetype metadata to know which nodes are queues/brokers and their consumer-group shape).

**In scope**
- New calculation module (pure math, no engine rewrite) reads arrival rate + service rate + replica count for queue/broker archetypes.
- New scoring/warning output: "this queue has no way to keep up with configured traffic," "no redundant consumer," etc.
- Surfaced in the existing results UI (Score/Sim tab), not a new tab.

**Size:** S–M. Self-contained math module + a handful of new rule checks.

**Tasks**
1. Implement the M/M/1 and M/M/c formulas (utilization, expected queue length, wait time, stability check ρ = λ/μ) as a pure, dependency-free module.
2. Wire inputs: arrival rate from existing simulator QPS output, service rate from `maxQPS` per replica, `c` from replica count.
3. Add a distinct warning for ρ ≥ 1 (unbounded backlog) — separate from today's "critical utilization" status, since the failure mode is different (instability vs. just being busy).
4. Add a warning for broker/queue-archetype nodes with no redundant consumer (single consumer, no consumer group).
5. Wire the new warnings into `SimulationResult.warnings` (or a new field) so they render in the existing Score/Sim tab — no new tab.
6. Decide and document whether these feed an existing scoring category (e.g. availability/scalability) or stay advisory-only for phase 1 — must not break the exactly-20-points-per-category invariant.
7. Add fixture designs: at least one known-unstable queue (confirm it's flagged) and one stable one (confirm no false positive).

---

## P2 — EPIC-04: Language/framework limitation rules

**Goal.** Encode known real-world limits (e.g. Node.js is single-core without clustering) as a new metadata dimension + rule evaluator, same shape as existing scoring rules.

**Depends on:** EPIC-01 (same metadata/rule-engine pattern).

**In scope**
- `runtimeModel` (or similar) field on relevant component specs: single-threaded / multi-threaded / managed-pool, etc.
- Rule(s) that flag mismatches (CPU-bound load on a single-core runtime with no clustering configured).
- Content accuracy check against CLAUDE.md's "every formula/attribution must be correct" invariant — this is exactly the kind of claim that needs a real source, not a guess.

**Size:** S once the archetype/metadata pattern from EPIC-01 exists; mostly content research + a few rules.

**Tasks**
1. Research phase: compile a sourced, accurate list of real language/runtime limitations relevant to components already in the library (e.g. Node.js single-core without `cluster`/`worker_threads`, Python GIL) — must clear CLAUDE.md's "every attribution must be correct" bar.
2. Add a `runtimeModel` (or similar) metadata field to relevant component specs.
3. Tag existing components with the field where it applies.
4. Write rule(s) cross-referencing `runtimeModel` against configured load/replica/scaling settings, emitting a warning on mismatch.
5. Write explanatory copy per warning (why it's real, not just a flag) — same pattern as existing `conceptLibrary.ts` teaching content.
6. Accuracy review pass on all new claims before merge.

---

## P2 — EPIC-05: Codegen scaffolding via LLM

**Goal.** Generate structured scaffolding (docker-compose/IaC snippets, API stubs matching declared connections, a precise architecture prompt) from the graph, via a small backend proxy to a local Ollama model or Azure AI Foundry.

**Depends on:** EPIC-01 + EPIC-02 (needs a stable, semantically rich graph model to generate anything meaningful from).

**In scope**
- One backend endpoint (the only backend surface in phase 1) proxying to the configured LLM.
- Output scoped to **scaffolding + prompt generation**, not "produce a working deployed app" — set user-facing expectations accordingly (see feasibility doc §4 risk note on small-model reliability).
- Graph → structured description serialization (this doubles as the input both to the LLM and to the "hand this prompt to your own coding assistant" path).

**Out of scope:** the app itself running/deploying generated code.

**Size:** M–L. New surface from scratch (no existing groundwork per the codebase exploration); scope discipline is the main risk, not the engineering.

**Tasks**
1. Design the graph → structured-description serialization (components w/ archetypes, connections w/ semantics, capacity settings).
2. Stand up the single backend proxy endpoint forwarding to local Ollama or Azure AI Foundry.
3. Write the prompt template(s) turning the serialized graph into scaffolding output (docker-compose/API stubs, or a plain descriptive prompt for external use).
4. Build the "generate" UI action (button/panel) calling the endpoint and displaying/downloading output.
5. Add explicit UI messaging on scope ("scaffolding + prompt, not a finished deployed app") to manage expectations.
6. Handle LLM endpoint config/secrets (Ollama URL or Foundry key) as the one deliberate backend exception in an otherwise client-side app.
7. Manual QA pass with the real ~2GB-RAM local model to validate actual output quality before calling this done — this is the epic's main product risk, not a formality.

---

## P3 (deferred) — EPIC-06: Discrete-event simulation + live playback

**Goal.** Real time-stepped simulation: queues that visibly fill/drain, nodes that go down and recover, per-archetype event-driven behavior (Kafka-log replay vs SQS-style delete-on-ack vs pub/sub fanout timing).

**Depends on:** EPIC-01, 02, 03 (reuses the same archetypes/edges; queueing math becomes the sanity-check baseline for DES output).

**Not started.** Tracked so the phase-1 work (esp. EPIC-01's archetype design and the versioned `SimulationResult` shape) doesn't foreclose it. Algorithm: standard discrete-event simulation (simulated clock + timestamped event priority queue) — well-understood, feasible fully client-side, but effectively a second engine. Biggest single item in the whole vision; do not start until phase 1 has shipped and been used.

**Tasks (sketch only — revisit and re-scope before starting)**
1. Design the event/clock core (priority queue + simulated clock) as a module separate from `engine/simulator.ts`, not a modification of it.
2. Design one event-driven state machine per archetype from EPIC-01 (log-replay, delete-on-ack queue, pub/sub fanout timing, node up/down + recovery).
3. Design a playback/animation layer decoupled from computation (compute the event log fast, animate it over wall-clock time separately).
4. Define a new versioned output shape alongside `SimulationResult` so the existing Score tab keeps working unmodified.

---

## P3 (deferred) — EPIC-07: Cloud persistence / accounts

**Goal.** Move from client-side-only `localStorage` to accounts + shared/saved designs server-side.

**Depends on:** persistence already being called through an abstraction (largely true today via `safeStorage.ts` — verify it cleanly supports an async/networked backend before assuming this is a small lift).

**Not started.** No architectural decision needed now beyond "don't call `localStorage` directly outside the existing storage abstraction" — already the current pattern per CLAUDE.md.

**Tasks (sketch only — revisit and re-scope before starting)**
1. Audit `safeStorage.ts` and every persisted store for sync-only assumptions that would block an async, network-backed storage engine.
2. Design the account/auth model.
3. Design the shared/multi-user design data model.

---

## Suggested execution order

EPIC-01 → (EPIC-02, EPIC-03 in parallel) → EPIC-04 → EPIC-05 → *ship, use it for a while* → EPIC-06 / EPIC-07 as their own later push.
