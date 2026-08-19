## D1: Audit page — keep DSL table, revise issue acceptance criteria

**Choice:** Keep the existing 30-line pages-ui DSL audit table. The `blocks-audit-trail-viewer` is ledger-specific and unsuitable for IoT bridge audit events. Issue #95 acceptance criteria ("Audit page uses `<blocks-audit-trail-viewer>`") should be revised — the assumption that the component is general-purpose was incorrect.
**Alternatives:**
- Build a new `<blocks-event-trail>` component — premature abstraction; the audit-trail-viewer's reusable residue (table + filters + expandable details) is already provided by `pages-table`
- Add `mode="event-log"` to existing `blocks-audit-trail-viewer` — couples unrelated domains (ledger crypto-verification vs IoT event logging) in one component via mode switch; every future feature must consider both modes
- Adapt data to ledger shape — forces wrong domain model (`BridgeAuditEvent` into `LedgerEntry` with fake digests and sequence numbers)
**Rationale:** Code inspection of the 695-line `audit-trail-viewer.ts` confirms it is saturated with ledger-specific concerns at every layer: `LedgerEntry` data model with `digest`/`sequenceNumber`/`causedByEntryId`, verification banner (`treeRoot`/`verified`/`redactedCount`), attestation UI (`verdict`: SOUND/FLAGGED/ENDORSED/CHALLENGED), and hardcoded `/api/v1/ledger/entries` endpoint construction. Stripping these yields a bare table with filters — functionality `pages-table` with `client-sort`, `client-filter`, and `detailMode="single"` already provides. The IoT bridge audit trail (`BridgeAuditEvent` with `eventType`/`deviceId`/`message`) is a third domain shape that maps to neither `LedgerEntry` nor a hypothetical generic event model.
**Trade-offs:** Audit page stays as DSL table without expandable row details. If richer audit viewing becomes a requirement, a page-specific `hostPanel()` component can be added later without the baggage of a generic abstraction.
**Sources:** `blocks-ui/components/audit-trail-viewer/src/audit-trail-viewer.ts` (695 lines, ledger-specific throughout), `webapp/src/main/webapp/src/pages/audit.ts` (30-line DSL table), issue #95 acceptance criteria
**Exploration:** quick → revised round 1
**Status:** revised — dropped blocks-audit-trail-viewer usage and new event-trail component; reviewer demonstrated the component is ledger-saturated with no extractable generic pattern

## D2: Data flow — hostPanel()-driven integration

**Choice:** Use `hostPanel()` DSL function for embedding blocks-ui web components in pages-ui pages. Components receive configuration through `configure(panelProps)` with full complex-property support, and optionally integrate with the pages-ui data pipeline via the `DataReceiver` interface and `lookup` binding.
**Alternatives:**
- `html()` string blocks — uses raw `innerHTML` (`el.innerHTML = props.content`); only supports string attributes, not complex properties (objects, arrays, functions); inadequate for components needing `WorkIdentity`, `MetricDefinition[]`, or callbacks
- Lit template blocks (extend DSL with `component()` returning `TemplateResult`) — new framework mechanism; `hostPanel()` already exists and solves the same problem
- DSL-native integration (`kpiRow()`, `auditTrail()` functions) — higher coupling; `hostPanel()` is the generic version of this
- Direct replacement — replace DSL pages entirely with raw Lit; abandons pages-ui sidebar/routing/dataset management
**Rationale:** `hostPanel()` is the platform's existing first-class mechanism for hosting external web components in pages-ui. The runtime: (1) looks up the tag name via `registerPanel(typeName, tagName)`, (2) calls `document.createElement(tagName)`, (3) calls `configure(panelProps)` before `connectedCallback()` — enabling complex property delivery, (4) optionally wires `DataReceiver` for pages-ui data pipeline integration via `lookup`. Both `KpiMetricRow.configure()` and `AuditTrailViewer.configure()` already implement the `ConfigurablePanel` interface. This was the most consequential implicit decision in the spec — defaulting to `html()` was incorrect and constrained all downstream decisions.
**Trade-offs:** Requires `registerPanel()` calls in the IoT webapp to map type names to tag names. Components must implement `configure()` (existing blocks-ui components already do via `DataSourceMixin`).
**Sources:** `pages-component/src/model/hosting.ts` (`ConfigurablePanel`, `DataReceiver` interfaces), `pages-ui/src/dsl/builders.ts:547` (`hostPanel()` function), `pages-runtime/src/panel-registry.ts` (`registerPanel()`), `pages-runtime/src/activation-host-panel.test.ts` (integration tests proving `configure()` before `connectedCallback()`), `pages-runtime/src/content.ts:13` (`renderHtml` — `el.innerHTML = props.content`)
**Exploration:** quick → revised round 1
**Status:** revised — switched from html() to hostPanel(); reviewer correctly identified that html()/innerHTML cannot support complex property binding; investigation revealed hostPanel() as the existing proper mechanism

## D3: KPI data — new REST endpoints

**Choice:** Add `GET /api/devices/kpi` and `GET /api/health/kpi` REST endpoints that return `MetricDefinition[]` server-side. Server computes counts and status aggregations.
**Alternatives:**
- Client-side compute — fetch `/api/devices` then count/filter in TypeScript; this is what `device-kpi-row.ts` does today with `count("deviceId")`, `filterBy("available", ...)`, `distinct("providerId")` on the already-fetched `devices` dataset
**Rationale:** The `<blocks-kpi-metric-row>` component is endpoint-driven by design — it expects a `MetricDefinition[]` from a URL. With `hostPanel()`, the endpoint is passed via `configure({ endpoint: "/api/devices/kpi" })`. The server computes aggregated metrics as `COUNT(*)` / `GROUP BY` queries — cheaper than transmitting the full device list for client-side counting. These are different queries at different granularities serving different consumers: `/api/devices` returns the full list for table display; `/api/devices/kpi` returns 3-4 compact metric objects. This is not a double-fetch — it's appropriately granular API design.
**Trade-offs:** New Java REST endpoints to maintain. The current client-side approach in `device-kpi-row.ts` (27 lines of DSL with `count()`/`distinct()`/`filterBy()`) works at current scale — server endpoints become essential only when device counts grow or trends/sparklines are added. MetricDefinition shape couples Java response to blocks-ui TypeScript interface (existing platform pattern per `LedgerEntry`).
**Sources:** `blocks-ui/components/kpi-metric-row/src/kpi-metric-row.ts` (`MetricDefinition` interface lines 10-17, `_fetchMetrics()` lines 240-254), `webapp/src/main/webapp/src/components/device-kpi-row.ts` (27-line DSL using client-side aggregations)
**Exploration:** quick → revised round 1
**Status:** revised — rationale corrected to focus on endpoint-driven component design and server-side aggregation efficiency; removed speculative trend/sparkline justification

## D4: Overall approach — mostly IoT-only with hostPanel() integration

**Choice:** Three-step phased integration, mostly IoT-only. One minor blocks-ui change (adding `configure()` to `blocks-detail-pane`) is the only cross-repo work — not a component design phase. Sequence: (1) KPI rows via `hostPanel("kpi-metric-row")` — validates the integration lifecycle; (2) work items via `masterDetail()` DSL function with `hostPanel("work-item-detail")` as detail — uses existing DSL layout composition, not `blocks-split-workbench`; (3) case/device detail sub-pages via `hostPanel("detail-pane")` — requires adding `configure()` to `blocks-detail-pane` first (5-line method).
**Alternatives:**
- `hostPanel("split-workbench")` for work items — `SplitWorkbench` lacks `configure()` and is a slot-based layout component; `masterDetail()` DSL function already composes `split()` + `data-table` + `host-panel` and is the correct mechanism
- All-at-once — misses the opportunity to validate the integration pattern incrementally
- Defer tabbed detail pages — avoid the blocks-ui change entirely; case/device detail stays as-is until `detail-pane` gains `configure()`
**Rationale:** `masterDetail()` at `pages-ui/src/dsl/builders.ts:558` composes a split layout from DSL primitives: `masterDetail({ master: table({...}), detail: hostPanel("work-item-detail", {...}) })`. This uses the work-item-detail component (which HAS `configure()`) for the detail side and a DSL data-table for the master side. No `blocks-split-workbench` needed — it's a slot-based layout component without `configure()` that serves a different composition pattern (imperative Lit children vs declarative DSL). `blocks-list-pane` extends `DataSourceMixin` and inherits `configure()`, but is also unnecessary when using the DSL data-table as master. `blocks-detail-pane` genuinely lacks `configure()` — adding it is a 5-line method (`tabs`, `selectionTopic`, `emptyMessage` property assignment). Since `hostPanel()` passes `panelProps` by reference (no serialization), function callbacks like `TabDefinition.badge` survive. KPI rows go first (simplest, all primitives), work items second (validates `masterDetail()` + complex `configure()` props), tabbed detail third (after the minor blocks-ui change).
**Trade-offs:** One blocks-ui change required (adding `configure()` to `blocks-detail-pane`), but this is a single method addition — not the component design phase of the original D4. Audit page remains as DSL table per D1. Issue #95 acceptance criteria should be updated: work items page uses `masterDetail()` DSL with `blocks-work-item-detail`, not `blocks-split-workbench`.
**Sources:** Issue #95 acceptance criteria, `pages-ui/src/dsl/builders.ts:558` (`masterDetail()`), `split-workbench/src/split-workbench.ts` (no `configure()`), `list-pane/src/list-pane.ts` (has `configure()` via `DataSourceMixin`), `detail-pane/src/detail-pane.ts` (no `configure()`), `pages-component/src/controller/data-source-mixin.ts:51` (`configure()`)
**Exploration:** quick → revised round 1 → revised round 2
**Status:** revised — step 2 uses `masterDetail()` DSL (not `blocks-split-workbench`); step 3 requires adding `configure()` to `blocks-detail-pane` (minor blocks-ui change); corrected reviewer's error about `list-pane` (it has `configure()` via DataSourceMixin)

## D5: Refresh coordination — independent lifecycles accepted

**Choice:** Accept independent refresh lifecycles between pages-ui datasets and blocks-ui endpoint-driven components. No explicit coordination mechanism.
**Alternatives:**
- Coordinated refresh — synchronize pages-ui dataset refresh with blocks-ui component refresh via shared event bus or SSE triggers; adds coupling for marginal UX benefit
- Single data source — route all data through pages-ui datasets using `DataReceiver`; requires blocks-ui components to abandon endpoint self-sufficiency
**Rationale:** With `hostPanel()` + endpoint-driven components, the KPI row fetches from `/api/devices/kpi` while the table's `devices` dataset fetches from `/api/devices`. Both are REST-polled on independent intervals (the `sse://api/devices/stream` dataset is a separate event stream, not the device list backing the table). Slight temporal inconsistency between KPI counts and table rows (measured in seconds within the longer polling interval) is acceptable — both converge on the next refresh cycle. The alternative — coordinating refresh across two REST polling cycles — adds architectural coupling disproportionate to the UX concern.
**Trade-offs:** Brief visual inconsistency between metrics and table data during refresh windows. If this becomes user-visible, SSE-driven refresh triggers can be added without changing the component architecture.
**Sources:** `pages-component/src/controller/data-source-mixin.ts` (`DataSourceMixin` endpoint sync), `blocks-ui-core` `DataSourceAdapter` (independent refresh lifecycle)
**Exploration:** new — surfaced by reviewer in R1-16
**Status:** captured
