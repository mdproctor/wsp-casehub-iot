# Bridge Audit Trail Viewer Design

## Problem

The IoT bridge audit page is a 34-line pages-ui DSL table (`audit.ts`) with dropdown filters for event type and device, date range pickers, sortable columns, and CSV export. It works for flat scanning but cannot show the rich nested payloads inside `BridgeMessage` (7 sealed variants: StateChange, StateSnapshot, ProviderStatusChange, Command, CommandResponse, Heartbeat, ReplayedStateChange). Operators cannot expand a row to see what a state change actually changed, trace a command to its response via `correlationId`, or distinguish protocol lifecycle events (AGENT_CONNECTED/DISCONNECTED) from data-bearing events.

Issue #103 identified four revisit triggers: expandable row details, filtering, correlation tracking, and export/reporting. The `pages-event-trail` platform component in pages-ui-components covers the first three natively and, with targeted upstream fixes, the fourth.

## Architecture

### Tab-toggled dual view

The audit page becomes a `tabs()` DSL layout with two tabs:

- **Table** — the existing DSL table. Fast flat scanning, CSV export, dropdown filters. Zero changes to the current implementation except extracting filters to page level.
- **Trail** — `pages-event-trail` hosted via `hostPanel("event-trail", { ... })`. Expandable row details with tailored rendering per `BridgeMessage` variant, chip filtering, and correlation display.

Both tabs share common filter state (deviceId, eventType, date range) so an operator filtering to a specific device in Table mode keeps that filter when switching to Trail for expanded details.

```typescript
// audit.ts — revised structure
import { page, rows, tabs, panel, table, columns, selector,
         datePicker, lookup, groupBy, col, sortBy, hostPanel } from "@casehubio/pages-ui";

export function auditPage() {
  return page("Audit",
    rows(
      // Shared filters — page level, above tabs
      columns([2, 2, 2, 6],
        [selector({
          title: "Event Type",
          filter: { enabled: true, group: "audit" },
          lookup: lookup("audit", groupBy("eventType", col("eventType"))),
          subtype: "dropdown",
        })],
        [selector({
          title: "Device",
          filter: { enabled: true, group: "audit" },
          lookup: lookup("audit", groupBy("deviceId", col("deviceId"))),
          subtype: "dropdown",
        })],
        [datePicker({ field: "dateFrom", label: "From Date" })],
        [datePicker({ field: "dateTo", label: "To Date" })],
      ),

      // Tab-toggled views
      tabs(
        ["Table", panel("Audit Trail", table({
          title: "Event History",
          sortable: true,
          pageSize: 25,
          csvExport: true,
          filter: { listening: true, group: "audit" },
          lookup: lookup("audit", sortBy("timestamp", "DESCENDING")),
        }))],
        ["Trail", hostPanel("event-trail", {
          endpoint: "/api/bridge/audit",
          recordsPath: "records",
          chipField: "eventType",
          chipValues: [
            "STATE_CHANGE", "COMMAND_SENT", "COMMAND_RESPONSE",
            "STATE_SNAPSHOT", "PROVIDER_STATUS_CHANGE",
            "AGENT_CONNECTED", "AGENT_DISCONNECTED",
            "REPLAYED_STATE_CHANGE",
          ],
          entityField: "deviceId",
          entityLabel: "Device",
          showDateRange: true,
          csvExport: true,
          getRowDetail: renderAuditDetail,
          getRowKey: (row) => `${row.get("occurredAt")}-${row.get("eventType")}`,
        })],
      ),
    ),
  );
}
```

### Shared filter architecture

The page-level filter bar (selectors + date pickers) operates on the `"audit"` filter group. The Table tab's `table()` already listens to this group via `filter: { listening: true, group: "audit" }`.

For the Trail tab, `pages-event-trail` uses its own built-in filter bar (chips, entity selector, date range) rather than the DSL filter group. The shared filter state is achieved by passing the active filter values as URL parameters to the Trail's endpoint. When page-level filters change, the Trail's endpoint URL is reconstructed with the current filter params, and `pages-event-trail` refetches.

The Trail's built-in chip and entity filters provide additional refinement within the shared scope — an operator can filter to deviceId=X at page level, then further refine by eventType chips in the Trail view.

## Upstream changes — casehub-pages

Three targeted fixes to `pages-event-trail.ts` in `pages-ui-components`. These harden the component for all `hostPanel()` consumers, not just IoT.

### 1. configure() property passthrough (~3 lines)

`PagesEventTrail.configure()` currently handles: `data`, `columnDefs`, `columnConfig`, `chipField`, `chipValues`, `entityField`, `entityLabel`, `showDateRange`. Properties not in this list — including `getRowDetail`, `getRowKey`, and `columnRenderers` — are silently dropped.

```typescript
// pages-event-trail.ts — configure() additions
override configure(props: Record<string, unknown>): void {
  // ... existing property handling ...
  if (props.getRowDetail !== undefined) this.getRowDetail = props.getRowDetail as (row: TypedRow) => TemplateResult | undefined;
  if (props.getRowKey !== undefined) this.getRowKey = props.getRowKey as (row: TypedRow) => string;
  if (props.columnRenderers !== undefined) this.columnRenderers = props.columnRenderers as ReadonlyMap<ColumnId, ColumnRenderer>;
  super.configure(props);
}
```

### 2. recordsPath for wrapped responses (~10 lines)

`createSourceFactory()` casts the fetch response as `unknown[]`. Endpoints that return wrapped responses (e.g. `AuditTrailView` with `{ records, totalCount, offset, limit }`) break silently.

Add a `recordsPath` property:

```typescript
@property({ type: String }) recordsPath?: string;

// In configure():
if (props.recordsPath !== undefined) this.recordsPath = props.recordsPath as string;

// In createSourceFactory(), after response.json():
.then((body: unknown) => {
  const entries = this.recordsPath
    ? (body as Record<string, unknown>)[this.recordsPath] as unknown[]
    : body as unknown[];
  // ... existing logic with entries ...
})
```

### 3. csvExport passthrough (~2 lines)

The internal `pages-table` does not receive `csvExport`. Add:

```typescript
@property({ type: Boolean }) csvExport = false;

// In render(), on the pages-table element:
?csvExport=${this.csvExport}

// In configure():
if (props.csvExport !== undefined) this.csvExport = props.csvExport as boolean;
```

## Detail rendering

### TypeScript type model

A TypeScript module defines discriminated union types mirroring the Java sealed interface. This enables compile-time exhaustiveness checking on `switch` statements.

```typescript
// types/bridge-audit.ts

interface StateChangePayload {
  "@type": "STATE_CHANGE";
  event: { deviceId: string; property: string; oldValue: unknown; newValue: unknown; timestamp: string };
}

interface ReplayedStateChangePayload {
  "@type": "REPLAYED_STATE_CHANGE";
  event: { deviceId: string; property: string; oldValue: unknown; newValue: unknown; timestamp: string };
}

interface StateSnapshotPayload {
  "@type": "STATE_SNAPSHOT";
  devices: Array<{ deviceId: string; deviceClass: string; available: boolean }>;
}

interface ProviderStatusPayload {
  "@type": "PROVIDER_STATUS";
  status: { providerId: string; state: string; message?: string };
}

interface CommandPayload {
  "@type": "COMMAND";
  correlationId: string;
  command: { deviceId: string; action: string; parameters?: Record<string, unknown> };
}

interface CommandResponsePayload {
  "@type": "COMMAND_RESULT";
  correlationId: string;
  result: { success: boolean; message?: string; error?: string };
}

interface HeartbeatPayload {
  "@type": "HEARTBEAT";
}

type BridgeMessagePayload =
  | StateChangePayload
  | ReplayedStateChangePayload
  | StateSnapshotPayload
  | ProviderStatusPayload
  | CommandPayload
  | CommandResponsePayload
  | HeartbeatPayload;

type BridgeAuditEventType =
  | "STATE_CHANGE"
  | "REPLAYED_STATE_CHANGE"
  | "STATE_SNAPSHOT"
  | "PROVIDER_STATUS_CHANGE"
  | "COMMAND_SENT"
  | "COMMAND_RESPONSE"
  | "AGENT_CONNECTED"
  | "AGENT_DISCONNECTED";

interface AuditRecord {
  eventType: BridgeAuditEventType;
  deviceId: string | null;
  correlationId: string | null;
  payload: BridgeMessagePayload | null;
  occurredAt: string;
}
```

### Rendering strategy

The `getRowDetail` callback branches on `eventType` first (always present), then on `payload.@type` for message-bearing events:

```typescript
// renderers/audit-detail.ts

function renderAuditDetail(row: TypedRow): TemplateResult | undefined {
  const eventType = row.get("eventType") as BridgeAuditEventType;
  const payload = row.get("payload") as BridgeMessagePayload | null;
  const correlationId = row.get("correlationId") as string | null;

  return html`
    <div class="audit-detail">
      ${renderPayload(eventType, payload)}
      ${correlationId ? renderCorrelation(correlationId) : nothing}
    </div>
  `;
}
```

`renderPayload` handles the dispatch:

| eventType | payload | Rendering |
|-----------|---------|-----------|
| STATE_CHANGE | StateChangePayload | Device ID, property name, old → new value, timestamp |
| REPLAYED_STATE_CHANGE | ReplayedStateChangePayload | Same as STATE_CHANGE with "Replayed" badge |
| STATE_SNAPSHOT | StateSnapshotPayload | Device count, list of devices with class and availability |
| PROVIDER_STATUS_CHANGE | ProviderStatusPayload | Provider ID, state, message |
| COMMAND_SENT | CommandPayload | Target device, action, parameters table |
| COMMAND_RESPONSE | CommandResponsePayload | Success/failure badge, message or error |
| AGENT_CONNECTED | null | "Bridge agent connected" with timestamp |
| AGENT_DISCONNECTED | null | "Bridge agent disconnected" with timestamp |

StateChange and ReplayedStateChange share a renderer, differing only in a badge.

## Correlation display

When a row with a non-null `correlationId` is expanded, the detail pane renders the payload (above) plus a "Related Events" section. Related events are fetched via a targeted server-side query:

```
GET /api/bridge/audit?correlationId={correlationId}
```

This uses the existing `BridgeAuditQuery.correlationId` filter — no new endpoint needed. The query returns the complete correlation group regardless of the main view's pagination state (fixing the cross-page boundary issue identified in decision review R1-13).

Typical correlation groups are 2-4 events:
- `COMMAND_SENT` → `COMMAND_RESPONSE` (command round-trip)
- `COMMAND_SENT` → `STATE_CHANGE` → `COMMAND_RESPONSE` (command with side-effect)

The "Related Events" section renders each sibling as a compact row: timestamp, eventType badge, and a one-line summary extracted from the payload. The current row is highlighted to orient the operator within the chain.

The fetch is triggered on expand (not pre-loaded) and cached client-side per correlationId for the duration of the page session to avoid re-fetching on collapse/re-expand.

## Component registration

The IoT webapp registers the panel type at startup:

```typescript
// app.ts — after imports
import '@casehubio/pages-ui-components/event-trail';
registerPanel("event-trail", "pages-event-trail");
```

This makes `hostPanel("event-trail", { ... })` resolve to the `<pages-event-trail>` custom element.

## Data flow

```
                    ┌─────────────────────────────────────────────┐
                    │              Audit Page                      │
                    │  ┌─────────────────────────────────────┐    │
                    │  │  Shared Filters (page level)         │    │
                    │  │  eventType │ deviceId │ dateFrom/To  │    │
                    │  └──────────────┬────────────────────────┘    │
                    │                 │ filter params                │
                    │    ┌────────────┴────────────┐                │
                    │    ▼                         ▼                │
                    │  ┌──────────┐          ┌──────────┐          │
                    │  │  Table   │          │  Trail   │          │
                    │  │  tab     │          │  tab     │          │
                    │  │          │          │          │          │
                    │  │ DSL table│          │ pages-   │          │
                    │  │ via      │          │ event-   │          │
                    │  │ dataset  │          │ trail    │          │
                    │  │ "audit"  │          │ via      │          │
                    │  │          │          │ hostPanel│          │
                    │  └────┬─────┘          └────┬─────┘          │
                    │       │                      │                │
                    └───────┼──────────────────────┼────────────────┘
                            │                      │
                            ▼                      ▼
                    GET /api/bridge/audit    GET /api/bridge/audit
                    (via dataset pipeline)  (via PagesEventTrail
                                             createSourceFactory)
                            │                      │
                            └──────────┬───────────┘
                                       ▼
                              BridgeAuditStore.query()
```

On row expand with correlationId:

```
Trail tab → expand row → correlationId != null
  → GET /api/bridge/audit?correlationId=X
  → render payload + related events in detail pane
```

## Testing strategy

### Unit tests (TypeScript)

- **Type model tests**: Verify discriminated union exhaustiveness — a test that switches on all `@type` values fails to compile if a variant is missing.
- **Renderer tests**: Each `BridgeMessage` variant has a test that passes a fixture payload and asserts the rendered HTML contains expected field values. Null-message events (AGENT_CONNECTED/DISCONNECTED) render without NPE.
- **Correlation fetch**: Mock fetch to verify `?correlationId=X` is called on expand, results are rendered, and caching prevents re-fetch on collapse/re-expand.

### Integration tests (Quarkus)

- **Tab rendering**: The audit page renders both tabs, switching between them preserves page structure.
- **Shared filters**: Applying a filter in Table tab and switching to Trail tab shows the filter is active.
- **Detail expand**: Expanding a trail row renders the tailored detail pane for each event type.
- **Correlation display**: Expanding a row with a correlationId shows related events from the server response.

### Existing coverage

- `BridgeAuditStore` implementations (JPA, in-memory) have query tests including correlationId filtering.
- `BridgeAuditEvent` and `BridgeMessage` serialization tests exist.
- The REST endpoint `getBridgeAudit` is covered by existing webapp tests.

## Scope boundaries

**In scope:**
- Tab-toggled audit page with shared filters
- Trail view via pages-event-trail + hostPanel
- Tailored detail rendering per BridgeMessage variant
- TypeScript type model for BridgeMessage
- Correlation display via server-side query
- Upstream pages-event-trail fixes (configure, recordsPath, csvExport)

**Out of scope:**
- Pagination for the Trail view — PagesEventTrail loads up to the endpoint's limit (100 default, 500 max). Adequate for current scale; pagination support is a separate upstream enhancement if needed at production volume.
- Severity filtering — `BridgeAuditEvent` has no severity field; this was a hypothetical trigger in #103 that doesn't map to the data model.
- Real-time SSE streaming for the Trail view — the Table tab already has this via the dataset pipeline; adding it to PagesEventTrail is a separate enhancement.

## Implementation phases

### Phase 1: Upstream fixes (casehub-pages)

File issue against casehub-pages. Implement the three targeted fixes to `pages-event-trail.ts`:
1. configure() property passthrough (getRowDetail, getRowKey, columnRenderers)
2. recordsPath for wrapped responses
3. csvExport passthrough

Publish SNAPSHOT. Update IoT webapp's pages dependency.

### Phase 2: TypeScript types and renderers (IoT webapp)

1. Create `types/bridge-audit.ts` with discriminated union types
2. Create `renderers/audit-detail.ts` with per-eventType rendering functions
3. Unit test each renderer with fixture data

### Phase 3: Tab-toggled page (IoT webapp)

1. Register `pages-event-trail` panel in `app.ts`
2. Refactor `audit.ts`: extract filters to page level, wrap Table and Trail in `tabs()`
3. Wire `getRowDetail`, `getRowKey`, column config to the Trail hostPanel

### Phase 4: Correlation (IoT webapp)

1. Implement correlation fetch in `renderAuditDetail` — on expand, query `/api/bridge/audit?correlationId=X`
2. Render "Related Events" section in the detail pane
3. Add client-side caching per correlationId

## References

- `webapp/src/main/webapp/src/pages/audit.ts` — current 34-line DSL table
- `webapp/src/main/webapp/src/app.ts:33` — dataset("audit", "/api/bridge/audit")
- `pages-ui-components/src/event-trail/pages-event-trail.ts` — platform event trail component (gaps at configure:91, createSourceFactory:40)
- `pages-ui/src/dsl/builders.ts:245` — tabs() DSL function
- `pages-ui/src/dsl/builders.ts:574` — hostPanel() DSL function
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java` — 7 sealed variants
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEvent.java` — record with @Nullable BridgeMessage
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java` — 8 event types (2 null-message)
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditQuery.java` — query with correlationId support
- `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTOperationsApi.java:94` — getBridgeAudit REST endpoint
- `webapp-api/src/main/java/io/casehub/iot/webapp/view/AuditTrailView.java` — response shape { records, totalCount, offset, limit }
- #95 D1 — rationale for keeping DSL table over blocks-audit-trail-viewer
- #95 D2 — hostPanel() integration pattern validation
- Review R1-02 — configure() does not deliver getRowDetail/getRowKey/columnRenderers
- Review R1-03 — data shape mismatch (endpoint returns wrapped object, component expects array)
- Review R1-06 — CSV export not passed through to internal pages-table
- Review R1-09/R1-18 — TypeScript exhaustiveness requires discriminated union types
- Review R1-10 — AGENT_CONNECTED/DISCONNECTED have null message payload
- Review R1-13 — client-side correlation fails across page boundaries
- Review R1-16 — filter state loss defeats dual-view drill-down workflow
