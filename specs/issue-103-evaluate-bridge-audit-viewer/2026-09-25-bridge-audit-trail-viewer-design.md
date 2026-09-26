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
          endpoint: "/api/bridge/audit?limit=500",
          recordsPath: "records",
          columnDefs: [
            { id: "eventType", name: "Event", type: ColumnType.LABEL, getValue: (r: any) => r.eventType },
            { id: "deviceId", name: "Device", type: ColumnType.TEXT, getValue: (r: any) => r.deviceId },
            { id: "correlationId", name: "Correlation", type: ColumnType.TEXT, getValue: (r: any) => r.correlationId },
            { id: "occurredAt", name: "Time", type: ColumnType.DATE, getValue: (r: any) => r.occurredAt },
          ],
          csvExport: true,
          getRowDetail: renderAuditDetail,
          getRowKey: auditRowKey,
        })],
      ),
    ),
  );
}
```

### Shared filter architecture

The page-level filter bar (selectors + date pickers) sits above the tabs. Both tabs share the same filter state via different synchronisation mechanisms.

**Table tab:** Listens to the `"audit"` filter group via `filter: { listening: true, group: "audit" }` — standard DSL dataset pipeline.

**Trail tab:** A page-level filter adapter translates filter group changes into server-side query parameters via URL parameter injection. When any page-level selector or date picker changes:

1. Read current `eventType`, `deviceId`, `dateFrom`, `dateTo` from the filter group state
2. Construct the Trail's endpoint URL: `/api/bridge/audit?limit=500` with non-null filter values appended as query params (`&eventType=STATE_CHANGE&deviceId=sw-1&from=...&to=...`)
3. Update the Trail panel's endpoint and call `syncEndpoint()` to trigger a re-fetch with the new URL

The `getBridgeAudit` REST endpoint already accepts `eventType`, `deviceId`, `from`, and `to` query params — this is server-side filtering, no new endpoint logic needed. PagesEventTrail's `resolveEndpoint()` preserves existing query params when appending date range.

**No duplicate filter UI:** The Trail's built-in chip bar, entity selector, and date range picker are not configured (`chipField`, `entityField`, `showDateRange` omitted from hostPanel config). The page-level selectors are the sole filter controls for both tabs. This eliminates the confusing duplicate filter bars where page-level and Trail-level filters compete.

**Adapter implementation:** The filter adapter is page-level TypeScript code (~20 lines) that:
1. **On filter group change:** reads current filter values, constructs the endpoint URL, and calls `syncEndpoint()` on the Trail panel via DOM access.
2. **On Trail tab activation:** applies current filter state immediately (initial sync). If the operator sets filters in Table tab before switching to Trail, the adapter runs on first Trail activation — not just on subsequent filter changes — ensuring the Trail loads with the correct filter params from the start.

No additional upstream changes beyond the four listed are required — the adapter constructs the full URL page-side.

## Upstream changes — casehub-pages

Four targeted fixes to `pages-event-trail.ts` in `pages-ui-components`. These harden the component for all `hostPanel()` consumers, not just IoT.

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

### 4. Raw entry pass-through for getRowDetail (~15 lines)

The `pages-data` pipeline converts raw entries to `TypedRow` via `fromRows()`. `TypedRow` cells support only scalar `CellValue` types (TEXT, NUMBER, DATE, LABEL, NULL) — complex objects like `BridgeMessage` are destroyed (converted to `"[object Object]"` via `String()`). Detail renderers that need access to the raw entry (nested payloads, polymorphic objects) cannot retrieve them from `TypedRow`.

Fix: `pages-event-trail` maintains a `WeakMap<TypedRow, unknown>` mapping each `TypedRow` to its corresponding raw entry. The map is built during `_applyFilters()` using index correspondence (the `i`-th row in the filtered `TypedDataSet` corresponds to the `i`-th element in the filtered raw entries array). The `getRowDetail` callback is wrapped before being passed to `pages-table` to inject the raw entry as an optional second parameter.

```typescript
// pages-event-trail.ts additions
@state() private _rawEntryByRow = new WeakMap<TypedRow, unknown>();

// In _applyFilters(), after building _filteredDataSet:
this._rawEntryByRow = new WeakMap();
for (let i = 0; i < this._filteredDataSet.rows.length; i++) {
  const row = this._filteredDataSet.rows[i];
  if (row) this._rawEntryByRow.set(row, filtered[i]);
}

// In render(), wrap getRowDetail:
const wrappedDetail = this.getRowDetail
  ? (row: TypedRow) => this.getRowDetail!(row, this._rawEntryByRow.get(row))
  : undefined;
// ... then on pages-table:
.getRowDetail=${wrappedDetail}

// Updated property type (non-breaking — second param optional):
@property({ type: Object }) getRowDetail?: (row: TypedRow, rawEntry?: unknown) => TemplateResult | undefined;
```

This is non-breaking: existing consumers that define `getRowDetail(row)` with one parameter work unchanged. Consumers needing raw data opt in via the second parameter.

## Detail rendering

### TypeScript type model

A TypeScript module defines discriminated union types derived from the actual Java sealed interface and Jackson serialization output. Each `BridgeMessage` variant carries `tenancyId` and `timestamp` (from the sealed interface contract) plus variant-specific fields. The `@type` discriminator follows Jackson's `@JsonTypeInfo(property = "@type")` convention.

`DeviceEntity` is polymorphic — Jackson serializes it with a `@deviceType` discriminator and all instance fields (base + subclass-specific). The TypeScript model uses an index signature for type-specific fields (brightness, temperature, etc.) which the renderer accesses dynamically via `changedCapabilities`.

```typescript
// types/bridge-audit.ts

interface DeviceEntity {
  "@deviceType": string;
  deviceId: string;
  deviceClass: string;
  label: string;
  available: boolean;
  lastUpdated: string;
  tenancyId: string;
  providerId: string;
  location: string | null;
  [key: string]: unknown;
}

interface StateChangeEvent {
  before: DeviceEntity | null;
  after: DeviceEntity;
  changedCapabilities: string[];
  occurredAt: string;
  providerId: string;
}

interface StateChangePayload {
  "@type": "STATE_CHANGE";
  tenancyId: string;
  timestamp: string;
  event: StateChangeEvent;
}

interface ReplayedStateChangePayload {
  "@type": "REPLAYED_STATE_CHANGE";
  tenancyId: string;
  timestamp: string;
  event: StateChangeEvent;
}

interface StateSnapshotPayload {
  "@type": "STATE_SNAPSHOT";
  tenancyId: string;
  timestamp: string;
  devices: DeviceEntity[];
}

interface ProviderStatusPayload {
  "@type": "PROVIDER_STATUS";
  tenancyId: string;
  timestamp: string;
  status: {
    providerId: string;
    previousStatus: "CONNECTED" | "CONNECTING" | "DISCONNECTED";
    currentStatus: "CONNECTED" | "CONNECTING" | "DISCONNECTED";
  };
}

interface CommandPayload {
  "@type": "COMMAND";
  tenancyId: string;
  timestamp: string;
  correlationId: string;
  command: {
    targetDeviceId: string;
    action: string;
    parameters: Record<string, unknown>;
    dispatchedBy: string;
    correlationId: string;
  };
}

interface CommandResponsePayload {
  "@type": "COMMAND_RESULT";
  tenancyId: string;
  timestamp: string;
  correlationId: string;
  result: "SENT" | "FAILED" | "TIMEOUT";
}

type BridgeMessagePayload =
  | StateChangePayload
  | ReplayedStateChangePayload
  | StateSnapshotPayload
  | ProviderStatusPayload
  | CommandPayload
  | CommandResponsePayload;

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

### Row key derivation

The `getRowKey` function uses `TypedRow` cell accessors (not the fictional `get()` method):

```typescript
// renderers/audit-detail.ts

function auditRowKey(row: TypedRow): string {
  const at = row.date("occurredAt" as ColumnId).toISOString();
  const type = row.text("eventType" as ColumnId);
  const devCell = row.cell("deviceId" as ColumnId);
  const dev = devCell.type === "NULL" ? "system" : (devCell as { value: string }).value;
  return `${at}-${type}-${dev}`;
}
```

### Rendering strategy

The `getRowDetail` callback receives the raw `AuditRecord` as an optional second parameter (via upstream change #4). This bypasses the `TypedRow` cell pipeline entirely for detail rendering — the `payload` field (a complex `BridgeMessage` object with nested polymorphic types) would be destroyed by the `CellValue` conversion (which only supports TEXT, NUMBER, DATE, LABEL, NULL).

```typescript
// renderers/audit-detail.ts

function renderAuditDetail(row: TypedRow, rawEntry?: unknown): TemplateResult | undefined {
  const record = rawEntry as AuditRecord | undefined;
  if (!record) return undefined;

  return html`
    <div class="audit-detail">
      ${renderPayload(record.eventType, record.payload)}
      ${record.correlationId ? renderCorrelation(record.correlationId) : nothing}
    </div>
  `;
}
```

`renderPayload` handles the dispatch:

| eventType | payload | Rendering |
|-----------|---------|-----------|
| STATE_CHANGE | StateChangePayload | Device label + ID from `event.after`, capability diff table: `event.before[cap] → event.after[cap]` for each `cap` in `event.changedCapabilities` |
| REPLAYED_STATE_CHANGE | ReplayedStateChangePayload | Same as STATE_CHANGE with "Replayed" badge |
| STATE_SNAPSHOT | StateSnapshotPayload | Device count, compact list showing `@deviceType`, `deviceId`, `label`, `available` per device |
| PROVIDER_STATUS_CHANGE | ProviderStatusPayload | Provider ID, `status.previousStatus → status.currentStatus` with transition badges |
| COMMAND_SENT | CommandPayload | `command.targetDeviceId`, `command.action`, parameters key-value table, `command.dispatchedBy` |
| COMMAND_RESPONSE | CommandResponsePayload | Result badge: SENT (green), FAILED (red), TIMEOUT (amber) |
| AGENT_CONNECTED | null | "Bridge agent connected" with timestamp |
| AGENT_DISCONNECTED | null | "Bridge agent disconnected" with timestamp |

StateChange and ReplayedStateChange share a renderer, differing only in a badge. The capability diff renderer uses `changedCapabilities` to identify which type-specific fields changed, then reads the before/after values from the polymorphic `DeviceEntity` objects via dynamic property access.

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

### Async pattern for correlation fetch

The `getRowDetail` callback is synchronous (`(row: TypedRow) => TemplateResult | undefined`). Correlation data requires an async fetch. This is handled via Lit's `until()` directive, which renders a fallback while the Promise is in-flight and re-renders with the resolved value automatically — no explicit re-render trigger needed.

```typescript
import { until } from 'lit/directives/until.js';

function renderCorrelation(correlationId: string): TemplateResult {
  const promise = fetchCorrelation(correlationId)
    .then(events => html`
      <div class="related-events">
        <h4>Related Events</h4>
        ${events.map(renderRelatedRow)}
      </div>
    `)
    .catch(() => html`<span class="error">Failed to load related events</span>`);

  return html`${until(promise, html`<span class="loading">Loading related events...</span>`)}`;
}
```

**Caching:** A module-scope `Map<string, Promise<AuditRecord[]>>` caches the fetch Promise per correlationId. Subsequent expand/collapse/re-expand of the same correlation group returns the cached Promise without re-fetching. The cache persists until page navigation (audit events are historical — no invalidation needed within a page session).

```typescript
const correlationCache = new Map<string, Promise<AuditRecord[]>>();

function fetchCorrelation(correlationId: string): Promise<AuditRecord[]> {
  if (!correlationCache.has(correlationId)) {
    correlationCache.set(correlationId,
      fetch(`/api/bridge/audit?correlationId=${correlationId}`)
        .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
        .then(body => body.records)
        .catch(err => {
          correlationCache.delete(correlationId);
          throw err;
        })
    );
  }
  return correlationCache.get(correlationId)!;
}
```

**States:** Loading (until fallback shown), success (related events rendered, cached), error (error message shown via `.catch()`, cache entry deleted so next expand retries).

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
- Upstream pages-event-trail fixes (configure, recordsPath, csvExport, raw entry pass-through)

**Out of scope:**
- Pagination for the Trail view — the Trail endpoint is configured with `limit=500` (the maximum). At typical bridge event rates this covers a meaningful time window. Pagination support is a separate upstream enhancement if needed at production volume.
- `totalCount` fix — `DefaultIoTOperationsApi.getBridgeAudit` returns `events.size()` (page size) as `totalCount`, not the actual total count of matching records. This is a backend bug that prevents "Showing X of Y" display and future pagination. Fix requires either a `count()` method on `BridgeAuditStore` or a wrapper result type from `query()`. Filed as a separate issue — does not block this spec since no UI element depends on `totalCount`.
- Severity filtering — `BridgeAuditEvent` has no severity field; this was a hypothetical trigger in #103 that doesn't map to the data model.
- Real-time SSE streaming for the Trail view — the Table tab already has this via the dataset pipeline; adding it to PagesEventTrail is a separate enhancement.

## Implementation phases

### Phase 0: Upstream issue filing (spec deliverable)

File issue against casehub-pages documenting the four upstream fixes as cross-repo dependencies. This is a spec deliverable, not deferred to implementation — the issue documents the contract and enables planning.

### Phase 1: Upstream fixes (casehub-pages)

Implement the four targeted fixes to `pages-event-trail.ts`:
1. configure() property passthrough (getRowDetail, getRowKey, columnRenderers)
2. recordsPath for wrapped responses
3. csvExport passthrough
4. Raw entry pass-through for getRowDetail (WeakMap-based TypedRow → raw entry mapping)

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
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditStore.java` — SPI interface (implemented: JPA, in-memory, no-op)
- `bridge-persistence-jpa/` — `JpaBridgeAuditStore` production implementation
- `bridge-persistence-memory/` — `InMemoryBridgeAuditStore` for testing
- `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTOperationsApi.java:94` — getBridgeAudit REST endpoint
- `webapp-api/src/main/java/io/casehub/iot/webapp/view/AuditTrailView.java` — response shape { records, totalCount, offset, limit }
- #35 — BridgeAuditStore SPI (CLOSED — fully implemented; ARC42STORIES §8 reference is stale and should be updated)
- #95 D1 — rationale for keeping DSL table over blocks-audit-trail-viewer
- #95 D2 — hostPanel() integration pattern validation
- Review R1-02 — configure() does not deliver getRowDetail/getRowKey/columnRenderers
- Review R1-03 — data shape mismatch (endpoint returns wrapped object, component expects array)
- Review R1-06 — CSV export not passed through to internal pages-table
- Review R1-09/R1-18 — TypeScript exhaustiveness requires discriminated union types
- Review R1-10 — AGENT_CONNECTED/DISCONNECTED have null message payload
- Review R1-13 — client-side correlation fails across page boundaries
- Review R1-16 — filter state loss defeats dual-view drill-down workflow
