# Showcase View — Design (EPIC-00)

Status: approved design, ready for implementation planning. Independent of the platform-evolution feasibility track (`docs/superpowers/specs/2026-08-27-systemforge-platform-feasibility.md`) and its epics (`docs/backlog.md` EPIC-01..07) — this ships real architectures the user has already deployed, not the engine's own evolution, and has no dependency on the archetype/connection-semantics work.

## 1. Motivation

The user wants a portfolio/showcase section inside SystemForge to present architectures he's actually deployed — starting with "DTE Masters," a real multi-tenant SaaS. This needs to be embeddable, standalone, into `mercally.com` (a separate Angular site he owns, same domain family) inside an "arquitecturas" section, as an iframe.

## 2. Scope

**In scope**
- A generic, reusable showcase system (not a one-off DTE Masters page) — adding a second/third architecture later means dropping in data, not writing code.
- A gallery view listing all showcased architectures.
- A standalone, embeddable, read-only view per architecture.
- The ~12 real components DTE Masters needs, split into generic archetypes (instance-labeled) and cloud-provider-specific specs.
- One small core-engine addition: an optional per-instance node description field (see §5) — a real editor capability, not a showcase-only hack.
- DTE Masters authored by Claude as a static JSON file checked into the repo (not drawn by the user in this pass).

**Out of scope**
- Deployment/hosting setup itself (not deployed yet; this design just avoids blocking it).
- Responsive iframe auto-height (nice-to-have, not required for v1 — fixed/scrollable iframe is acceptable to start).
- Any interview-mode features (Score/Sim/Capacity/Tradeoffs) in the showcase view — this is a portfolio piece, not an exercise.
- Editing/authoring UI for showcase JSON (author by hand for now).

## 3. Routes & UI

- `/showcase` — gallery. Grid of cards, one per architecture (title, short description), visual language matching the existing `ProblemSelector` pattern. Each card links to `/showcase/[slug]`.
- `/showcase/[slug]` — the embeddable view. A minimal shell: **only** the ReactFlow canvas in `readOnly` mode (reusing the existing gating already used by reference-solution tabs) — no sidebar, no component palette, no tab bar, no interview bar. Clicking a node opens a small side panel showing its `label` and `description` (see §5). No score/capacity/tradeoffs UI.
- Both routes are new additions to the App Router (today the app has a single `/` route, `src/app/page.tsx`).

## 4. Data model

**Showcase design file** — one JSON per architecture, e.g. `data/showcase/dte-masters.json`:

```
{
  "slug": "dte-masters",
  "title": "DTE Masters",
  "description": "...",
  "nodes": [
    { "componentId": "compute-host", "x": 100, "y": 100, "label": "VPS (Docker + Traefik)", "description": "..." },
    { "componentId": "payment-gateway", "x": 300, "y": 100, "label": "Wompi", "description": "Pasarela de pago, webhook a frontend+backend" }
  ],
  "edges": [
    { "source": "<node-array-index-or-id>", "target": "<...>" }
  ]
}
```

`componentId` may reference either the main component library (`data/components.ts` — e.g. reuse `sql-db` for the external Postgres provider) or the new showcase-only component pool (§4b). `label`/`description` are per-instance overrides using the **same fields** the core editor already has (`label`) or is gaining (`description`, §5) — no showcase-specific schema fork.

A `data/showcase/index.ts` lists all showcase slugs for the gallery to enumerate (add one line per new architecture).

**New component pool** — `data/showcaseComponents.ts`, separate from `data/components.ts` (per the earlier decision: vendor-specific components don't belong in the interview-prep palette, and don't need `conceptLibrary.ts`/`learningPath.ts` entries since those invariants are scoped to the main library only). Two categories:

- **Generic archetypes** (label overridden per instance to name the real product): `payment-gateway` (→ "Wompi"), `background-job-processor` (→ "Hangfire"), `reverse-proxy` (→ "Traefik"), `compute-host` (→ "VPS (Docker)").
- **Cloud-provider-specific** (distinct identity per service, since multi-cloud differentiation is architecturally meaningful): `azure-blob-storage`, `azure-key-vault`, `azure-app-service`, `azure-app-registration`, `microsoft-graph`, `gcp-artifact-registry`, `gcp-pubsub`, `gmail-api`.

`getComponentById`-equivalent lookup for the showcase view checks the main library first, then this pool — mirrors the existing `customComponentsStore` fallback pattern already in `data/components.ts`.

## 5. Core-engine addition: per-instance node description

Today `ComponentNodeData` carries `label` (instance-level override, already used by reference solutions) but no `description` at the instance level — only `SystemComponent.description` at the spec level, which is fixed and shared across every instance of that component type.

Add an optional `description?: string` to `ComponentNodeData` (`store/canvasStore.ts`), and a corresponding text field in the node Props tab (`RightPanel.tsx`) so **any** user, in **any** design, can annotate a node instance — not a showcase-only field. The showcase click-panel (§3) reads this same field. No parallel schema.

## 6. Embed mechanics

- `next.config.ts` gains a header rule scoping `Content-Security-Policy: frame-ancestors https://mercally.com` to the `/showcase/:path*` route only — the main editor (`/`) stays un-embeddable, this is deliberately scoped.
- Real domain/subdomain to allow-list is confirmed at deploy time (not deployed yet); design just needs the header rule to exist and be easy to update with the final domain.
- No responsive auto-height in v1 — mercally.com's iframe uses a fixed height (or its own scroll), can be revisited later with a `postMessage`-based resize if it becomes annoying in practice.

## 7. Extensibility (adding architecture #2, #3...)

1. Add any missing component specs to `data/showcaseComponents.ts` (or reuse existing ones from either pool).
2. Write the new `data/showcase/<slug>.json`.
3. Add one line to `data/showcase/index.ts`.

No route/code changes needed per new architecture — this was the explicit requirement (generic system from day one, not a DTE-Masters-only page).

## 8. Risks / limitations

- Static JSON authored by hand (by Claude, per this pass) has no validation UI — a malformed file fails silently or ugly until someone opens `/showcase/[slug]`. Low risk at this scale (one architecture initially), worth a lightweight schema check (e.g. zod) if the count grows.
- `frame-ancestors` domain is a placeholder until real hosting/domain is decided — must be updated before the mercally.com embed actually works in production.
- Node positions (`x`/`y`) are hand-picked by Claude without the visual feedback a human gets dragging in the editor — first pass may need manual layout tweaks after review (still much faster than writing a renderer from scratch).

## Tasks

1. Add `description?: string` to `ComponentNodeData` (`store/canvasStore.ts`) + wire the field through node creation/update paths.
2. Add the description text input to the node Props tab (`components/panel/RightPanel.tsx` or its Props sub-tab).
3. Create `data/showcaseComponents.ts` with the 4 generic archetypes + 8 cloud-provider-specific specs (12 total), each with the same shape as `SystemComponent` (reusing `maxQPS`/`latencyMs`/`scalable`/`stateful` with reasonable representative values — accuracy of these numbers doesn't matter for a static showcase, visual correctness does).
4. Build the showcase-lookup helper (checks `data/components.ts` then `data/showcaseComponents.ts`) analogous to the existing `customComponentsStore` fallback in `getComponentById`.
5. Author `data/showcase/dte-masters.json`: model the ~9 real pieces of DTE Masters infra as nodes (VPS host possibly split into sub-nodes for frontend/backend/websocket/mailing/Hangfire, or kept as one annotated node — decide during authoring based on what reads clearest), wire the real connections (webhooks, Microsoft Graph email ingestion, Wompi webhook, GCP Pub/Sub + Gmail API), captioned via `label`/`description`.
6. Create `data/showcase/index.ts` listing the one entry.
7. Build `/showcase` gallery route: grid of cards from the index, visual pattern borrowed from `ProblemSelector`.
8. Build `/showcase/[slug]` route: minimal shell — canvas only, `readOnly` ReactFlow render (reuse the existing mechanism used by reference-solution tabs), node-click side panel showing `label`/`description`.
9. Add the `next.config.ts` header rule scoping `frame-ancestors` to the `/showcase/:path*` path (placeholder domain, flagged to update at deploy time).
10. Manual QA: open `/showcase/dte-masters` standalone, confirm no editor chrome leaks in, confirm it renders correctly inside a local `<iframe>` test page (simulating the mercally.com embed) before considering this done.
