# Replace Manual Webapp UI with blocks-ui Components

**Issue:** casehubio/iot#95
**Date:** 2026-08-18
**Status:** Design (revised after standard decision review — 3 rounds, 19 issues, APPROVED)

## Summary

Replace manual pages-ui DSL construction in the IoT webapp with blocks-ui web components using the platform's existing `hostPanel()` integration mechanism. Three page areas affected: KPI metric rows, work items, and device detail sub-pages. New REST endpoints provide server-computed KPI metrics. The audit page stays as-is — the blocks-ui audit-trail-viewer is ledger-specific and unsuitable for IoT bridge events.

## Scope

**In scope:**
- Add `configure()` to `blocks-detail-pane` in blocks-ui (5-line method — only cross-repo change)
- New `KpiResource` REST endpoints in IoT webapp (`/api/devices/kpi`, `/api/health/kpi`)
- Rewrite `devices.ts`, `health.ts`, `workitems.ts` to use `hostPanel()` with blocks-ui components
- Add `<blocks-detail-pane>` for device detail sub-page via `hostPanel()`
- Add `registerPanel()` calls in `app.ts` for component registration
- Delete `device-kpi-row.ts` (replaced by `<blocks-kpi-metric-row>`)

**Out of scope:**
- Audit page — stays as 30-line pages-ui DSL table (no blocks-ui match; see D1)
- `situations.ts`, `providers.ts`, `cases.ts` — low complexity, no blocks-ui match
- New generic event-trail component (dropped after review — no extractable generic pattern)
- Refactoring `blocks-audit-trail-viewer` (ledger-specific, not related)

## Architecture

### Integration Mechanism: `hostPanel()`

`hostPanel()` is the platform's existing first-class mechanism for hosting external web components in pages-ui pages. The runtime:

1. Looks up the tag name via `registerPanel(typeName, tagName)`
2. Calls `document.createElement(tagName)`
3. Calls `configure(panelProps)` before `connectedCallback()` — enabling complex property delivery (objects, arrays, functions)
4. Optionally wires `DataReceiver` for pages-ui data pipeline integration via `lookup`

**Why not `html()`:** `html()` uses `innerHTML` which only supports string attributes. blocks-ui components need complex properties (`MetricDefinition[]`, `WorkIdentity`, `TabDefinition[]` with function callbacks). `hostPanel()` passes these by JS object reference with no serialization.

### Data Flow

blocks-ui components are endpoint-driven. Each component receives a REST endpoint URL via `configure({ endpoint })` and self-fetches its data. The pages-ui DSL page hosts the component; the component owns its own data lifecycle.

```
pages-ui DSL page
  └── hostPanel("kpi-metric-row", { endpoint: "/api/devices/kpi", columns: 4 })
        └── runtime calls registerPanel("kpi-metric-row", "blocks-kpi-metric-row")
        └── runtime calls document.createElement("blocks-kpi-metric-row")
        └── runtime calls el.configure({ endpoint: "/api/devices/kpi", columns: 4 })
        └── component self-fetches → JSON → render
```

## Component Registration

`app.ts` must register blocks-ui components before pages-ui can host them:

```typescript
import { registerPanel } from "@casehubio/pages-runtime";

// Register blocks-ui components for hostPanel() use
registerPanel("kpi-metric-row", "blocks-kpi-metric-row");
registerPanel("work-item-detail", "blocks-work-item-detail");
registerPanel("detail-pane", "blocks-detail-pane");

// Import the components to register custom element definitions
import "@casehubio/blocks-ui/kpi-metric-row";
import "@casehubio/blocks-ui/work-item-detail";
import "@casehubio/blocks-ui/detail-pane";
```

## blocks-ui Change: `configure()` on `blocks-detail-pane`

The only cross-repo change. `DetailPane` lacks the `configure()` method that `hostPanel()` calls. Adding it:

```typescript
configure(props: Record<string, unknown>): void {
  if (props.tabs !== undefined) this.tabs = props.tabs as TabDefinition[];
  if (props.selectionTopic !== undefined) this.selectionTopic = props.selectionTopic as string;
  if (props.emptyMessage !== undefined) this.emptyMessage = props.emptyMessage as string;
}
```

Since `hostPanel()` passes `panelProps` by JS object reference (no serialization), function callbacks like `TabDefinition.badge` survive.

## KPI REST Endpoints (IoT webapp — Java)

### `GET /api/devices/kpi`

Returns `KpiMetric[]` matching blocks-ui `MetricDefinition` shape by convention:

```java
public record KpiMetric(
    String key,
    Object value,
    String label,
    String unit,
    String status  // "normal" | "warning" | "critical"
) {}
```

Metrics computed server-side from `DeviceRegistry`:
- `total-devices` — count of all devices
- `online` — count where `available == true`, status `warning` if < 50%
- `providers` — count of distinct provider IDs
- `active-alerts` — count of active situations

### `GET /api/health/kpi`

Same `KpiMetric[]` shape. Metrics:
- `connected-providers` — count of providers with status CONNECTED
- `bridge-connections` — count from bridge connection registry
- `active-situations` — count of active situations
- `open-cases` — count of open cases

### Java Type Convention

`KpiMetric` is defined in `webapp-api` (Tier 1, no JPA). Field names match blocks-ui `MetricDefinition` by convention, not coupling. They evolve independently — if blocks-ui renames a field, the Java type is updated in a separate commit.

## Page Rewrites

### `devices.ts`

**KPI row — before:** `deviceKpiRow("devices")` → `columns([3,3,3,3], [metric(...)])` DSL
**KPI row — after:**
```typescript
hostPanel("kpi-metric-row", {
  endpoint: "/api/devices/kpi",
  columns: 4,
  density: "comfortable",
})
```

**Device detail sub-page — before:** inline metric cards + `panel("Actions", html(...))` + `panel("State History", table(...))`
**Device detail sub-page — after:** `hostPanel("detail-pane", { tabs, selectionTopic: "device" })` with three tabs:
- **State** — current device capabilities
- **History** — state change timeline
- **Actions** — command dispatch buttons

Delete `components/device-kpi-row.ts`.

### `health.ts`

**Before:** `columns([3,3,3,3], [panel("Providers", metric(...))])` with per-metric refresh
**After:**
```typescript
hostPanel("kpi-metric-row", {
  endpoint: "/api/health/kpi",
  columns: 4,
  density: "comfortable",
})
```

Provider status table stays as pages-ui DSL (simple table, no blocks-ui match).

### `workitems.ts`

**Before:** selector + table + inline HTML buttons (Claim, Approve, Reject)
**After:**
```typescript
masterDetail({
  master: table({
    title: "WorkItems",
    sortable: true,
    pageSize: 20,
    lookup: lookup("workitems", sortBy("createdAt", "DESCENDING")),
    refresh: { interval: 15000 },
    selection: "single",
  }),
  detail: hostPanel("work-item-detail", {
    endpoint: "/api",
    identity: { tenancyId: "default-tenant", userId: "operator" },
  }),
})
```

This replaces ~35 lines of DSL + inline HTML with a `masterDetail()` composition that provides full work item lifecycle management (claim, start, complete, reject, suspend, resume, cancel, release, delegate, escalate — all with proper dialogs).

### Audit page — no change

Stays as 30-line pages-ui DSL table. The `blocks-audit-trail-viewer` is saturated with ledger-specific concerns (Merkle verification, attestation display, `LedgerEntry` data model) and has no extractable generic pattern. The residue after stripping ledger specifics is a bare table with filters — functionality `pages-table` already provides.

## Phased Execution

### Phase 1: KPI rows (validates integration lifecycle)
1. Add `registerPanel("kpi-metric-row", "blocks-kpi-metric-row")` in `app.ts`
2. Add `KpiResource` Java REST endpoints
3. Rewrite `devices.ts` KPI row → `hostPanel("kpi-metric-row")`
4. Rewrite `health.ts` KPI row → same component, different endpoint
5. Delete `device-kpi-row.ts`
6. Verify: blocks-ui KPI cards render in dev mode with live data

### Phase 2: Work items (uses DSL layout composition)
1. Add `registerPanel("work-item-detail", "blocks-work-item-detail")` in `app.ts`
2. Rewrite `workitems.ts` → `masterDetail()` with `hostPanel("work-item-detail")`
3. Verify: split pane renders, work item selection shows detail, lifecycle actions work

### Phase 3: Device detail (requires blocks-ui change)
1. Add `configure()` to `blocks-detail-pane` in blocks-ui (5-line method)
2. Build + install blocks-ui SNAPSHOT
3. Add `registerPanel("detail-pane", "blocks-detail-pane")` in `app.ts`
4. Rewrite device detail sub-page → `hostPanel("detail-pane")`
5. Verify: tabbed detail renders with State, History, Actions tabs

## Test Plan

### blocks-ui (`blocks-detail-pane` — configure method)

| Test | Assertion |
|------|-----------|
| `configure()` sets tabs | Call `configure({ tabs })` → `tabs` property updated |
| `configure()` sets selectionTopic | Call `configure({ selectionTopic })` → attribute updated |
| Function callbacks survive configure | `TabDefinition.badge` function callable after `configure()` |

### IoT webapp (Java)

| Test | Assertion |
|------|-----------|
| `GET /api/devices/kpi` returns 4 metrics | Response is array with total-devices, online, providers, active-alerts |
| KPI status reflects availability | < 50% online → `warning` status on online metric |
| `GET /api/health/kpi` returns 4 metrics | Response is array with providers, bridge, situations, cases |
| Null-safe when no providers/situations | Empty registry → zero counts, no NPE |

### IoT webapp (TypeScript — visual verification)

| Test | Assertion |
|------|-----------|
| devices page renders KPI row | `<blocks-kpi-metric-row>` visible with 4 cards |
| health page renders KPI row | Same component, different endpoint, same visual |
| work items page renders master-detail | Split pane with list + detail, lifecycle actions |
| device detail has tabbed pane | Three tabs: State, History, Actions |
| `device-kpi-row.ts` deleted | File does not exist |
| audit page unchanged | Still renders as pages-ui DSL table |

## Garden Entries Referenced

- **GE-20260712-7250c5** — DataSourceMixin extraction issues (informs: `blocks-kpi-metric-row` bypasses DataSourceMixin with `_fetchMetrics()`, so no extraction pipeline issue for KPI rows)
- **GE-20260717-19540a** — esbuild TC39 decorator fix (ensure tsconfig has `experimentalDecorators: true`)
- **GE-20260717-c99f50** — TypedRow property access (relevant for `masterDetail()` table selection → `work-item-detail`)
- **GE-20260712-f5b872** — CSS custom properties cascade through shadow DOM (theme injection for embedded components)
- **GE-20260709-2084c9** — Vite dev + esbuild prod dual build HTML entry points

## References

- `pages-ui/src/dsl/builders.ts:547` — `hostPanel()` function
- `pages-ui/src/dsl/builders.ts:558` — `masterDetail()` function
- `pages-runtime/src/panel-registry.ts` — `registerPanel()`
- `pages-component/src/model/hosting.ts` — `ConfigurablePanel`, `DataReceiver` interfaces
- `blocks-ui/components/kpi-metric-row/src/kpi-metric-row.ts` — KPI component API, `configure()` at line 260
- `blocks-ui/components/work-item-detail/src/work-item-detail.ts` — work item lifecycle API, `configure()` at line 443
- `blocks-ui/components/detail-pane/src/detail-pane.ts` — tabbed detail API (missing `configure()`)
- `blocks-ui/components/audit-trail-viewer/src/audit-trail-viewer.ts` — 695 lines, ledger-saturated (excluded)
- `webapp/src/main/webapp/src/pages/*.ts` — current page implementations
- `webapp/src/main/webapp/src/components/device-kpi-row.ts` — component to be deleted
- Issue #95 — acceptance criteria (audit page criterion to be revised)
