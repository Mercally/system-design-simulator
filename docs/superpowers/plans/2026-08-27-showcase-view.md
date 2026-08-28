# Showcase View (EPIC-00) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a generic, reusable showcase system inside SystemForge that presents architectures the user has actually deployed — starting with "DTE Masters" — via a gallery route and a standalone, embeddable, read-only per-architecture route.

**Architecture:** Two new App Router routes (`/showcase`, `/showcase/[slug]`) driven by static JSON per architecture. A new, separate component-spec pool covers vendor/cloud-specific pieces DTE Masters needs, looked up through the existing `getComponentById`-then-fallback pattern already used by `customComponentsStore`. The embeddable route renders a bare, static `<ReactFlow>` (no `canvasStore`, no drag/connect logic) reusing the existing `nodeTypes` map so nodes look identical to the main editor. One small, genuinely general core-engine addition (a per-instance `description` field) is added along the way and reused by both the main editor's Props panel and the showcase's node-click panel.

**Tech Stack:** Next.js 16 (App Router) · React 19 · TypeScript · @xyflow/react v12 · Zustand v5 (existing stores, unmodified except the one type addition) · Tailwind v4.

**Spec:** `docs/superpowers/specs/2026-08-27-showcase-view-design.md` — read it alongside this plan; the plan implements it task-by-task.

## Global Constraints

- No unit test framework in this repo (see root `CLAUDE.md`) — every task's "test" step is `npx tsc --noEmit` (or `npm run build`, which also runs tsc) + `npm run lint`, plus an explicit manual browser check. There is no `pytest`/`jest` equivalent to write; do not introduce one.
- `nodeTypes`/`edgeTypes` passed to `<ReactFlow>` must be module-level constants, never created inline in a render function (CLAUDE.md invariant — an inline object causes full remounts). Task 5 reuses the existing `src/components/canvas/nodes/nodeTypes.ts` and `src/components/canvas/edges/edgeTypes.ts` exports directly.
- Dark theme only, zinc-* palette, cyan accent for interactive/selected state — match existing component styles exactly (copy classNames from the files referenced per task, don't invent a new palette).
- The showcase route must NOT import or depend on `useCanvasStore`, `useSimulationStore`, `useInterviewStore`, or any dialog/toast component — it is a static, standalone view. If a task's code pulls in one of these, that's a signal the task has scope-crept; stop and reconsider.
- Every new component spec (Task 2) needs a real `icon` name present in `ICON_MAP` (`src/lib/icons.ts`) — falling back to the default `Server` icon for everything defeats the point of a showcase meant to show real variety.
- Commit after each task (not after each step) — one task is one reviewable, working unit.

---

### Task 1: Per-instance node `description` (core engine addition)

**Files:**
- Modify: `src/store/canvasStore.ts:16-30` (`ComponentNodeData` interface)
- Modify: `src/components/panel/RightPanel.tsx:333-355` (Props tab, node properties block)

**Interfaces:**
- Produces: `ComponentNodeData.description?: string` — read by Task 5's `ShowcaseNodePanel` and by `RightPanel`'s own node properties block.

- [ ] **Step 1: Add the field to `ComponentNodeData`**

In `src/store/canvasStore.ts`, in the `ComponentNodeData` interface (currently lines 16-30), add the new optional field after `scalable`:

```ts
export interface ComponentNodeData {
  componentId: string;
  label: string;
  icon: string;
  category: string;
  replicas: number;
  maxQPS: number;
  latencyMs: number;
  scalable: boolean;
  description?: string;
  utilization?: number;
  status?: string;
  isBottleneck?: boolean;
  // ReactFlow v12 requires an index signature on custom node data types
  [key: string]: unknown;
}
```

- [ ] **Step 2: Add the editable field to the Props tab**

In `src/components/panel/RightPanel.tsx`, the node-properties block currently reads (lines 333-355):

```tsx
            {/* Replicas slider — shown for every component node */}
            <div>
              <div className="mb-1.5 flex items-center justify-between">
                <label className="text-xs text-zinc-400">Replicas</label>
                <span className="font-mono text-xs text-cyan-500">
                  {data.replicas as number}
                </span>
              </div>
              <Slider
                aria-label="Replicas"
                value={[data.replicas as number]}
                onValueChange={(v) =>
                  updateNodeData(selectedNode.id, { replicas: Array.isArray(v) ? v[0] : v })
                }
                min={1}
                max={20}
                step={1}
                className=""
              />
              <p className="mt-1 text-[11px] text-zinc-400">
                Effective capacity: {(data.maxQPS as number) === Infinity ? "∞" : new Intl.NumberFormat("en-US").format((data.maxQPS as number) * (data.replicas as number))} QPS
              </p>
            </div>

            {/* Info */}
```

Insert a new block between the replicas `<div>` and the `{/* Info */}` comment:

```tsx
            {/* Replicas slider — shown for every component node */}
            <div>
              <div className="mb-1.5 flex items-center justify-between">
                <label className="text-xs text-zinc-400">Replicas</label>
                <span className="font-mono text-xs text-cyan-500">
                  {data.replicas as number}
                </span>
              </div>
              <Slider
                aria-label="Replicas"
                value={[data.replicas as number]}
                onValueChange={(v) =>
                  updateNodeData(selectedNode.id, { replicas: Array.isArray(v) ? v[0] : v })
                }
                min={1}
                max={20}
                step={1}
                className=""
              />
              <p className="mt-1 text-[11px] text-zinc-400">
                Effective capacity: {(data.maxQPS as number) === Infinity ? "∞" : new Intl.NumberFormat("en-US").format((data.maxQPS as number) * (data.replicas as number))} QPS
              </p>
            </div>

            {/* Description — optional per-instance note */}
            <div>
              <label className="mb-1 block text-xs text-zinc-400">Description</label>
              <textarea
                value={(data.description as string) ?? ""}
                onChange={(e) =>
                  updateNodeData(selectedNode.id, { description: e.target.value })
                }
                placeholder="Optional note about this instance"
                rows={2}
                className="w-full resize-none rounded-md border border-zinc-700 bg-zinc-800 px-2.5 py-1.5 text-xs text-zinc-200 placeholder-zinc-500 outline-none focus:border-cyan-600 focus:ring-1 focus:ring-cyan-600/50"
              />
            </div>

            {/* Info */}
```

`updateNodeData` is already in scope in this file (used two lines above for `replicas`) — no new import needed.

- [ ] **Step 3: Type-check and build**

Run: `npx tsc --noEmit`
Expected: no errors.

Run: `npm run build`
Expected: build succeeds.

- [ ] **Step 4: Manual check**

Run: `npm run dev`, open `http://localhost:3000`, drag any component onto the canvas, select it, type text into the new "Description" field in the Props tab, click a different node then back — confirm the text persisted. Reload the page — confirm it survives (persisted via the existing `canvasStore` persist middleware, no changes needed there since `partialize` already spreads full node `data`).

- [ ] **Step 5: Commit**

```bash
git add src/store/canvasStore.ts src/components/panel/RightPanel.tsx
git commit -m "feat: add optional per-instance node description field"
```

---

### Task 2: Showcase component pool + icons

**Files:**
- Create: `src/data/showcaseComponents.ts`
- Create: `src/lib/getShowcaseComponentById.ts`
- Modify: `src/lib/icons.ts`

**Interfaces:**
- Produces: `SHOWCASE_COMPONENTS: SystemComponent[]` (exported from `showcaseComponents.ts`), `getShowcaseComponentById(id: string): SystemComponent | undefined` (exported from `getShowcaseComponentById.ts`) — consumed by Task 4's `buildShowcaseGraph`.
- Consumes: `SystemComponent`/`ComponentCategory` from `@/types/component`, `getComponentById` from `@/data/components`, `ICON_MAP` from `@/lib/icons`.

- [ ] **Step 1: Add missing icons to `ICON_MAP`**

`src/lib/icons.ts` currently imports and maps 25 icons (see file). Add these 8 new imports and map entries:

```ts
import {
  Globe,
  Cloudy,
  Network,
  Router,
  ShieldAlert,
  Server,
  KeyRound,
  Database,
  HardDrive,
  Zap,
  Archive,
  Search,
  MessageSquare,
  GitBranch,
  Activity,
  Radio,
  Clock,
  Waves,
  Bell,
  Share2,
  TrendingUp,
  Warehouse,
  Compass,
  Shield,
  Lock,
  ShieldOff,
  FolderOpen,
  ShieldCheck,
  Users,
  Box,
  CreditCard,
  Cog,
  Container,
  Key,
  Cloud,
  Mail,
  Package,
  Megaphone,
} from "lucide-react";

export const ICON_MAP: Record<string, React.ComponentType<{ className?: string }>> = {
  Globe,
  Cloudy,
  Network,
  Router,
  ShieldAlert,
  Server,
  KeyRound,
  Database,
  HardDrive,
  Zap,
  Archive,
  Search,
  MessageSquare,
  GitBranch,
  Activity,
  Radio,
  Clock,
  Waves,
  Bell,
  Share2,
  TrendingUp,
  Warehouse,
  Compass,
  Shield,
  Lock,
  ShieldOff,
  FolderOpen,
  ShieldCheck,
  Users,
  Box,
  CreditCard,
  Cog,
  Container,
  Key,
  Cloud,
  Mail,
  Package,
  Megaphone,
};
```

(`Megaphone` also fixes a pre-existing gap: the main library's `pub-sub` component already declares icon `"Megaphone"`, which had no entry here and was silently falling back to `Server`.)

- [ ] **Step 2: Create the showcase component pool**

Create `src/data/showcaseComponents.ts`:

```ts
import type { SystemComponent } from "@/types/component";

/**
 * Vendor/cloud-specific components used by showcase architectures — kept
 * separate from `data/components.ts` (the interview-prep library) since
 * these are real product names, not generic system-design primitives, and
 * don't need `conceptLibrary.ts`/`learningPath.ts` entries.
 */
export const SHOWCASE_COMPONENTS: SystemComponent[] = [
  // Generic archetypes — instance `label` is overridden per showcase node
  // to name the real product (e.g. "Wompi", "Hangfire").
  {
    id: "payment-gateway",
    label: "Payment Gateway",
    category: "networking",
    icon: "CreditCard",
    maxQPS: 2000,
    latencyMs: 150,
    scalable: true,
    stateful: false,
    description:
      "Third-party payment processing integration reached over HTTPS, typically paired with an inbound webhook for async payment confirmation.",
  },
  {
    id: "background-job-processor",
    label: "Background Job Processor",
    category: "compute",
    icon: "Cog",
    maxQPS: 500,
    latencyMs: 50,
    scalable: true,
    stateful: true,
    description:
      "Runs scheduled and fire-and-forget background jobs (retries, dashboards, recurring tasks) with job state persisted alongside the application.",
  },
  {
    id: "compute-host",
    label: "Compute Host",
    category: "compute",
    icon: "Container",
    maxQPS: 20000,
    latencyMs: 5,
    scalable: false,
    stateful: true,
    description:
      "A single self-managed host (VPS/VM) running multiple containerized services — scales vertically, not horizontally, unlike a managed compute service.",
  },
  // Cloud-provider-specific — kept distinct because multi-cloud choice is an
  // architecturally meaningful decision worth showing explicitly.
  {
    id: "azure-blob-storage",
    label: "Azure Blob Storage",
    category: "storage",
    icon: "Archive",
    maxQPS: 20000,
    latencyMs: 60,
    scalable: true,
    stateful: true,
    description:
      "Microsoft Azure's object storage service for unstructured data (documents, PDFs, generated files), accessed over HTTPS with account-key or Azure AD auth.",
  },
  {
    id: "azure-key-vault",
    label: "Azure Key Vault",
    category: "infrastructure",
    icon: "Key",
    maxQPS: 2000,
    latencyMs: 30,
    scalable: true,
    stateful: true,
    description:
      "Centralized secret/certificate/key management on Azure, typically scoped per environment (dev/staging/production) and accessed via managed identity.",
  },
  {
    id: "azure-app-service",
    label: "Azure App Service",
    category: "compute",
    icon: "Cloud",
    maxQPS: 5000,
    latencyMs: 25,
    scalable: true,
    stateful: false,
    description:
      "Azure's managed PaaS for hosting web apps/APIs without managing servers — used here to host a dedicated reporting API.",
  },
  {
    id: "azure-app-registration",
    label: "Azure App Registration",
    category: "infrastructure",
    icon: "ShieldCheck",
    maxQPS: 1000,
    latencyMs: 40,
    scalable: true,
    stateful: true,
    description:
      "Azure AD (Entra ID) application identity used for OAuth2/OIDC authentication and authorization across the platform's services.",
  },
  {
    id: "microsoft-graph",
    label: "Microsoft Graph",
    category: "networking",
    icon: "Mail",
    maxQPS: 1000,
    latencyMs: 200,
    scalable: true,
    stateful: false,
    description:
      "Microsoft's unified API for Microsoft 365 data — used here to read incoming email and receive webhook notifications for new messages.",
  },
  {
    id: "gcp-artifact-registry",
    label: "GCP Artifact Registry",
    category: "infrastructure",
    icon: "Package",
    maxQPS: 500,
    latencyMs: 100,
    scalable: true,
    stateful: true,
    description:
      "Google Cloud's managed container/artifact registry — stores built Docker images pulled by the deployment target.",
  },
  {
    id: "gcp-pubsub",
    label: "GCP Pub/Sub",
    category: "messaging",
    icon: "Megaphone",
    maxQPS: 50000,
    latencyMs: 10,
    scalable: true,
    stateful: true,
    description:
      "Google Cloud's managed pub/sub messaging service — used here to receive push notifications from the Gmail API watch subscription.",
  },
  {
    id: "gmail-api",
    label: "Gmail API",
    category: "networking",
    icon: "Mail",
    maxQPS: 250,
    latencyMs: 250,
    scalable: false,
    stateful: false,
    description:
      "Google's API for reading Gmail messages, used with a watch/push subscription (delivered via GCP Pub/Sub) instead of polling.",
  },
];
```

- [ ] **Step 3: Create the lookup helper**

Create `src/lib/getShowcaseComponentById.ts`:

```ts
import type { SystemComponent } from "@/types/component";
import { getComponentById } from "@/data/components";
import { SHOWCASE_COMPONENTS } from "@/data/showcaseComponents";

/**
 * Resolves a component spec for the showcase view: checks the main
 * interview-prep library first (so a showcase node can reuse e.g.
 * `sql-db` or `reverse-proxy`), then the showcase-only vendor pool.
 */
export function getShowcaseComponentById(id: string): SystemComponent | undefined {
  const main = getComponentById(id);
  if (main) return main;
  return SHOWCASE_COMPONENTS.find((c) => c.id === id);
}
```

- [ ] **Step 4: Type-check and build**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add src/lib/icons.ts src/data/showcaseComponents.ts src/lib/getShowcaseComponentById.ts
git commit -m "feat: add showcase-only component pool (cloud/vendor specs)"
```

---

### Task 3: Author DTE Masters showcase data

**Files:**
- Create: `src/data/showcase/dte-masters.json`
- Create: `src/data/showcase/index.ts`

**Interfaces:**
- Produces: the `ShowcaseDesign` JSON shape (documented inline below) and `SHOWCASE_INDEX: { slug: string; title: string; description: string }[]` — consumed by Task 6 (`/showcase/[slug]`) and Task 7 (`/showcase` gallery).

- [ ] **Step 1: Write `src/data/showcase/dte-masters.json`**

```json
{
  "slug": "dte-masters",
  "title": "DTE Masters",
  "description": "Multi-tenant SaaS that captures electronic invoices (DTE) received and sent via Microsoft, Gmail, and Yahoo email accounts — automating PDF/JSON extraction, bulk date-range downloads, and centralized digital filing for accounting firms and multi-business owners.",
  "nodes": [
    { "id": "vps-host", "componentId": "compute-host", "x": 0, "y": 240, "label": "VPS (Docker Host)", "description": "Single VPS running Docker; hosts Traefik and all app containers below." },
    { "id": "traefik", "componentId": "reverse-proxy", "x": 260, "y": 240, "label": "Traefik", "description": "Reverse proxy + TLS termination for the domain, routes to the containers below." },
    { "id": "frontend", "componentId": "app-server", "x": 520, "y": 80, "label": "Frontend (SPA)" },
    { "id": "backend", "componentId": "app-server", "x": 520, "y": 240, "label": "Backend API" },
    { "id": "websocket", "componentId": "websocket-server", "x": 520, "y": 400, "label": "WebSocket Server" },
    { "id": "mailing", "componentId": "notification-service", "x": 780, "y": 400, "label": "Mailing Service" },
    { "id": "hangfire", "componentId": "background-job-processor", "x": 780, "y": 560, "label": "Hangfire (Background Jobs)" },
    { "id": "postgres", "componentId": "sql-db", "x": 780, "y": 80, "label": "External Postgres Provider", "description": "Managed Postgres from a third-party provider, not self-hosted on the VPS." },
    { "id": "blob-storage", "componentId": "azure-blob-storage", "x": 1040, "y": 80, "label": "Azure Blob Storage", "description": "Stores extracted invoice PDFs/JSON files." },
    { "id": "key-vault", "componentId": "azure-key-vault", "x": 1040, "y": 240, "label": "Azure Key Vault", "description": "Separate vaults for dev/staging/production secrets." },
    { "id": "app-service-reporting", "componentId": "azure-app-service", "x": 1040, "y": 400, "label": "Azure App Service", "description": "Reporting API exposed via OData." },
    { "id": "app-registration", "componentId": "azure-app-registration", "x": 1300, "y": 400, "label": "Azure App Registration", "description": "Authentication/authorization for the platform." },
    { "id": "ms-graph", "componentId": "microsoft-graph", "x": 1300, "y": 80, "label": "Microsoft Graph", "description": "Reads invoice emails from connected Microsoft mailboxes; also delivers webhook notifications for new mail." },
    { "id": "gmail-api", "componentId": "gmail-api", "x": 1300, "y": 240, "label": "Gmail API", "description": "Reads invoice emails from connected Gmail mailboxes via a watch subscription." },
    { "id": "pubsub", "componentId": "gcp-pubsub", "x": 1560, "y": 240, "label": "GCP Pub/Sub", "description": "Delivers Gmail API watch push notifications back to the backend." },
    { "id": "artifact-registry", "componentId": "gcp-artifact-registry", "x": 0, "y": 80, "label": "GCP Artifact Registry", "description": "Stores built Docker images pulled by the VPS on deploy." },
    { "id": "wompi", "componentId": "payment-gateway", "x": 520, "y": -80, "label": "Wompi", "description": "Payment gateway for subscription billing; confirms payments via webhook." }
  ],
  "edges": [
    { "source": "artifact-registry", "target": "vps-host" },
    { "source": "vps-host", "target": "traefik" },
    { "source": "traefik", "target": "frontend" },
    { "source": "traefik", "target": "backend" },
    { "source": "traefik", "target": "websocket" },
    { "source": "frontend", "target": "backend" },
    { "source": "frontend", "target": "wompi" },
    { "source": "backend", "target": "mailing" },
    { "source": "backend", "target": "hangfire" },
    { "source": "backend", "target": "postgres" },
    { "source": "backend", "target": "blob-storage" },
    { "source": "backend", "target": "key-vault" },
    { "source": "backend", "target": "app-service-reporting" },
    { "source": "backend", "target": "app-registration" },
    { "source": "backend", "target": "ms-graph" },
    { "source": "ms-graph", "target": "backend" },
    { "source": "backend", "target": "gmail-api" },
    { "source": "gmail-api", "target": "pubsub" },
    { "source": "pubsub", "target": "backend" },
    { "source": "wompi", "target": "backend" }
  ]
}
```

- [ ] **Step 2: Write `src/data/showcase/index.ts`**

```ts
export interface ShowcaseIndexEntry {
  slug: string;
  title: string;
  description: string;
}

export const SHOWCASE_INDEX: ShowcaseIndexEntry[] = [
  {
    slug: "dte-masters",
    title: "DTE Masters",
    description:
      "Multi-tenant SaaS capturing electronic invoices from Microsoft, Gmail, and Yahoo email accounts.",
  },
];
```

- [ ] **Step 3: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors (the JSON file has no TS types to check yet — this step becomes meaningful once Task 4's loader types it; for now just confirm `index.ts` compiles).

- [ ] **Step 4: Commit**

```bash
git add src/data/showcase/dte-masters.json src/data/showcase/index.ts
git commit -m "feat: author DTE Masters showcase architecture data"
```

---

### Task 4: `buildShowcaseGraph` — JSON to ReactFlow nodes/edges

**Files:**
- Create: `src/lib/buildShowcaseGraph.ts`

**Interfaces:**
- Consumes: `getShowcaseComponentById` (Task 2), `ComponentNodeData` type (`@/store/canvasStore`).
- Produces: `ShowcaseDesign` type, `loadShowcaseDesign(slug: string): ShowcaseDesign | undefined`, `buildShowcaseGraph(design: ShowcaseDesign): { nodes: Node<ComponentNodeData>[]; edges: Edge[] }` — consumed by Task 5 (`ShowcaseCanvas`) and Task 6 (`/showcase/[slug]` route).

- [ ] **Step 1: Write the loader + graph builder**

Create `src/lib/buildShowcaseGraph.ts`:

```ts
import type { Node, Edge } from "@xyflow/react";
import type { ComponentNodeData } from "@/store/canvasStore";
import { getShowcaseComponentById } from "@/lib/getShowcaseComponentById";
import { SHOWCASE_INDEX } from "@/data/showcase";
import dteMasters from "@/data/showcase/dte-masters.json";

export interface ShowcaseNodeSpec {
  id: string;
  componentId: string;
  x: number;
  y: number;
  label?: string;
  description?: string;
}

export interface ShowcaseEdgeSpec {
  source: string;
  target: string;
}

export interface ShowcaseDesign {
  slug: string;
  title: string;
  description: string;
  nodes: ShowcaseNodeSpec[];
  edges: ShowcaseEdgeSpec[];
}

const SHOWCASE_DESIGNS: Record<string, ShowcaseDesign> = {
  "dte-masters": dteMasters as ShowcaseDesign,
};

export function loadShowcaseDesign(slug: string): ShowcaseDesign | undefined {
  return SHOWCASE_DESIGNS[slug];
}

export function listShowcaseDesigns() {
  return SHOWCASE_INDEX;
}

export function buildShowcaseGraph(design: ShowcaseDesign): {
  nodes: Node<ComponentNodeData>[];
  edges: Edge[];
} {
  const nodes: Node<ComponentNodeData>[] = [];

  for (const spec of design.nodes) {
    const comp = getShowcaseComponentById(spec.componentId);
    if (!comp) continue;

    nodes.push({
      id: spec.id,
      type: "component",
      position: { x: spec.x, y: spec.y },
      draggable: false,
      connectable: false,
      selectable: true,
      data: {
        componentId: comp.id,
        label: spec.label ?? comp.label,
        icon: comp.icon,
        category: comp.category,
        replicas: 1,
        maxQPS: comp.maxQPS,
        latencyMs: comp.latencyMs,
        scalable: comp.scalable,
        description: spec.description,
      },
    });
  }

  const edges: Edge[] = design.edges.map((e) => ({
    id: `e-${e.source}-${e.target}`,
    source: e.source,
    target: e.target,
    type: "animated",
  }));

  return { nodes, edges };
}
```

Note: importing the `.json` file directly requires `resolveJsonModule` (TypeScript default under Next.js's bundled `tsconfig.json` already enables this — no config change needed; verified by the type-check step below).

- [ ] **Step 2: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors. If it fails with "Cannot find module '@/data/showcase/dte-masters.json'" or a JSON-import error, add `"resolveJsonModule": true` to `tsconfig.json`'s `compilerOptions` and re-run.

- [ ] **Step 3: Commit**

```bash
git add src/lib/buildShowcaseGraph.ts
git commit -m "feat: add showcase JSON-to-ReactFlow graph builder"
```

---

### Task 5: `ShowcaseCanvas` + `ShowcaseNodePanel` (static read-only render)

**Files:**
- Create: `src/components/showcase/ShowcaseCanvas.tsx`
- Create: `src/components/showcase/ShowcaseNodePanel.tsx`

**Interfaces:**
- Consumes: `nodeTypes` (`@/components/canvas/nodes/nodeTypes`), `edgeTypes` (`@/components/canvas/edges/edgeTypes`), `ShowcaseDesign`/`buildShowcaseGraph` (Task 4).
- Produces: `<ShowcaseCanvas design={ShowcaseDesign} />` — consumed by Task 6's `/showcase/[slug]` page.

- [ ] **Step 1: Write `ShowcaseNodePanel`**

Create `src/components/showcase/ShowcaseNodePanel.tsx`:

```tsx
"use client";

import type { ComponentNodeData } from "@/store/canvasStore";
import type { Node } from "@xyflow/react";

interface ShowcaseNodePanelProps {
  node: Node<ComponentNodeData> | null;
  onClose: () => void;
}

export function ShowcaseNodePanel({ node, onClose }: ShowcaseNodePanelProps) {
  if (!node) return null;
  const data = node.data;

  return (
    <div className="absolute right-4 top-4 z-10 w-72 rounded-lg border border-zinc-700 bg-zinc-900 p-4 shadow-xl">
      <div className="mb-2 flex items-start justify-between gap-2">
        <p className="text-sm font-medium text-zinc-100">{data.label}</p>
        <button
          onClick={onClose}
          aria-label="Close"
          className="text-xs text-zinc-500 hover:text-zinc-300"
        >
          ✕
        </button>
      </div>
      <p className="mb-2 text-xs uppercase tracking-wider text-zinc-500">
        {data.category}
      </p>
      {data.description ? (
        <p className="text-xs leading-relaxed text-zinc-300">{data.description}</p>
      ) : (
        <p className="text-xs italic text-zinc-500">No description provided.</p>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Write `ShowcaseCanvas`**

Create `src/components/showcase/ShowcaseCanvas.tsx`:

```tsx
"use client";

import { useMemo, useState } from "react";
import {
  ReactFlow,
  ReactFlowProvider,
  Background,
  BackgroundVariant,
  Controls,
  type Node,
  type NodeMouseHandler,
} from "@xyflow/react";
import "@xyflow/react/dist/style.css";
import { nodeTypes } from "@/components/canvas/nodes/nodeTypes";
import { edgeTypes } from "@/components/canvas/edges/edgeTypes";
import type { ComponentNodeData } from "@/store/canvasStore";
import { buildShowcaseGraph, type ShowcaseDesign } from "@/lib/buildShowcaseGraph";
import { ShowcaseNodePanel } from "./ShowcaseNodePanel";

interface ShowcaseCanvasProps {
  design: ShowcaseDesign;
}

function ShowcaseCanvasInner({ design }: ShowcaseCanvasProps) {
  const { nodes, edges } = useMemo(() => buildShowcaseGraph(design), [design]);
  const [selected, setSelected] = useState<Node<ComponentNodeData> | null>(null);

  const onNodeClick: NodeMouseHandler = (_event, node) => {
    setSelected(node as Node<ComponentNodeData>);
  };

  return (
    <div className="relative h-full w-full bg-zinc-950">
      <ReactFlow
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        onNodeClick={onNodeClick}
        onPaneClick={() => setSelected(null)}
        nodesDraggable={false}
        nodesConnectable={false}
        elementsSelectable
        fitView
        fitViewOptions={{ padding: 0.2 }}
        proOptions={{ hideAttribution: true }}
      >
        <Background variant={BackgroundVariant.Dots} color="rgba(155,172,205,0.24)" />
        <Controls showInteractive={false} />
      </ReactFlow>
      <ShowcaseNodePanel node={selected} onClose={() => setSelected(null)} />
    </div>
  );
}

export function ShowcaseCanvas(props: ShowcaseCanvasProps) {
  return (
    <ReactFlowProvider>
      <ShowcaseCanvasInner {...props} />
    </ReactFlowProvider>
  );
}
```

- [ ] **Step 3: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add src/components/showcase/ShowcaseCanvas.tsx src/components/showcase/ShowcaseNodePanel.tsx
git commit -m "feat: add static read-only showcase canvas + node info panel"
```

---

### Task 6: `/showcase/[slug]` route (the embeddable view)

**Files:**
- Create: `src/app/showcase/[slug]/page.tsx`

**Interfaces:**
- Consumes: `loadShowcaseDesign` (Task 4), `ShowcaseCanvas` (Task 5).

- [ ] **Step 1: Write the route**

Create `src/app/showcase/[slug]/page.tsx`:

```tsx
"use client";

import { useParams } from "next/navigation";
import { loadShowcaseDesign } from "@/lib/buildShowcaseGraph";
import { ShowcaseCanvas } from "@/components/showcase/ShowcaseCanvas";

export default function ShowcaseDetailPage() {
  const params = useParams<{ slug: string }>();
  const design = loadShowcaseDesign(params.slug);

  if (!design) {
    return (
      <div className="flex h-dvh items-center justify-center bg-zinc-950 text-sm text-zinc-500">
        Architecture not found.
      </div>
    );
  }

  return (
    <div className="h-dvh w-full">
      <ShowcaseCanvas design={design} />
    </div>
  );
}
```

This route intentionally renders nothing but the canvas — no `AppShell`, no sidebar, no top bar, no interview machinery — so it is safe to embed in an iframe as-is.

- [ ] **Step 2: Type-check and build**

Run: `npx tsc --noEmit`
Expected: no errors.

Run: `npm run build`
Expected: build succeeds (this also confirms the new dynamic route compiles).

- [ ] **Step 3: Manual check**

Run: `npm run dev`, open `http://localhost:3000/showcase/dte-masters` — confirm: no sidebar/top bar/palette visible, the 17 DTE Masters nodes render with their real labels, connections match Task 3's edge list, clicking a node opens the info panel with its description, clicking empty canvas closes it. Also open `http://localhost:3000/showcase/nonexistent-slug` — confirm the "Architecture not found." fallback renders instead of crashing.

- [ ] **Step 4: Commit**

```bash
git add src/app/showcase/[slug]/page.tsx
git commit -m "feat: add /showcase/[slug] embeddable read-only route"
```

---

### Task 7: `/showcase` gallery route

**Files:**
- Create: `src/components/showcase/ShowcaseGallery.tsx`
- Create: `src/app/showcase/page.tsx`

**Interfaces:**
- Consumes: `listShowcaseDesigns` (Task 4).

- [ ] **Step 1: Write the gallery component**

Create `src/components/showcase/ShowcaseGallery.tsx`. Card visual language matches the existing `ProblemSelector` pattern (zinc-800/900 surfaces, cyan accent, rounded-md) but as a grid, since this is a top-level page rather than a sidebar list:

```tsx
"use client";

import Link from "next/link";
import { listShowcaseDesigns } from "@/lib/buildShowcaseGraph";

export function ShowcaseGallery() {
  const designs = listShowcaseDesigns();

  return (
    <div className="mx-auto max-w-4xl px-6 py-12">
      <h1 className="mb-1 text-lg font-semibold text-zinc-100">Architectures</h1>
      <p className="mb-8 text-sm text-zinc-400">
        Real systems, presented read-only.
      </p>

      {designs.length === 0 ? (
        <p className="text-sm text-zinc-500">Nothing showcased yet.</p>
      ) : (
        <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
          {designs.map((d) => (
            <Link
              key={d.slug}
              href={`/showcase/${d.slug}`}
              className="rounded-lg border border-zinc-800 bg-zinc-900 p-4 transition-colors hover:border-zinc-700 hover:bg-zinc-800"
            >
              <p className="text-sm font-medium text-zinc-100">{d.title}</p>
              <p className="mt-1 text-xs leading-relaxed text-zinc-400">
                {d.description}
              </p>
            </Link>
          ))}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Write the route**

Create `src/app/showcase/page.tsx`:

```tsx
"use client";

import { ShowcaseGallery } from "@/components/showcase/ShowcaseGallery";

export default function ShowcasePage() {
  return (
    <div className="h-dvh w-full overflow-y-auto bg-zinc-950">
      <ShowcaseGallery />
    </div>
  );
}
```

- [ ] **Step 3: Type-check and build**

Run: `npx tsc --noEmit`
Expected: no errors.

Run: `npm run build`
Expected: build succeeds.

- [ ] **Step 4: Manual check**

Run: `npm run dev`, open `http://localhost:3000/showcase` — confirm the DTE Masters card renders with title + description, clicking it navigates to `/showcase/dte-masters` and shows the canvas from Task 6.

- [ ] **Step 5: Commit**

```bash
git add src/components/showcase/ShowcaseGallery.tsx src/app/showcase/page.tsx
git commit -m "feat: add /showcase gallery route"
```

---

### Task 8: CSP header for embedding

**Files:**
- Modify: `next.config.ts`

**Interfaces:** none (build/deploy config only).

- [ ] **Step 1: Add the headers() function**

`next.config.ts` currently reads:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  devIndicators: false,
};

export default nextConfig;
```

Replace with:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  devIndicators: false,
  async headers() {
    return [
      {
        // Scoped to the showcase route only — the main editor (`/`) stays
        // un-embeddable. Domain is a placeholder until real hosting is
        // decided (see docs/superpowers/specs/2026-08-27-showcase-view-design.md §6).
        source: "/showcase/:path*",
        headers: [
          {
            key: "Content-Security-Policy",
            value: "frame-ancestors 'self' https://mercally.com",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

- [ ] **Step 2: Build**

Run: `npm run build`
Expected: build succeeds.

- [ ] **Step 3: Manual check**

Run: `npm run start` (production server, headers only apply there — `next dev` does not reliably emit custom headers), then:

Run: `curl -I http://localhost:3000/showcase/dte-masters`
Expected: response includes `content-security-policy: frame-ancestors 'self' https://mercally.com`.

Run: `curl -I http://localhost:3000/`
Expected: no `content-security-policy` header (confirms the scoping to `/showcase/:path*` only).

- [ ] **Step 4: Commit**

```bash
git add next.config.ts
git commit -m "feat: scope CSP frame-ancestors header to /showcase for embedding"
```

---

### Task 9: End-to-end manual QA (standalone + iframe)

**Files:** none created/modified — verification only.

- [ ] **Step 1: Standalone check**

Run: `npm run build && npm run start`. Open `http://localhost:3000/showcase` and `http://localhost:3000/showcase/dte-masters` directly. Confirm: no console errors, no editor chrome (sidebar/top bar/tabs) visible anywhere on either route, all 17 nodes and their connections render correctly, node click/close works.

- [ ] **Step 2: Iframe embed check**

Create a throwaway local file (not part of the repo — e.g. in the scratchpad directory) named `iframe-test.html`:

```html
<!doctype html>
<html>
  <body style="margin:0">
    <iframe
      src="http://localhost:3000/showcase/dte-masters"
      style="width:100%;height:100vh;border:0"
    ></iframe>
  </body>
</html>
```

Open it directly in a browser (`file://` path). Confirm the canvas renders inside the iframe with no `X-Frame-Options`/CSP console error blocking it, and node click/close still works inside the embedded frame.

- [ ] **Step 3: Report**

No commit for this task — it's verification-only. If either check fails, return to the task whose deliverable is implicated (routing → Task 6/7, headers → Task 8) and fix before considering EPIC-00 done.
