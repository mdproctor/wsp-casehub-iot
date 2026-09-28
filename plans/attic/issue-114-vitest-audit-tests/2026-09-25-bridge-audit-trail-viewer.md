# Bridge Audit Trail Viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Evaluate richer bridge audit trail viewer
**Issue group:** #103

**Goal:** Replace the flat DSL audit table with a tab-toggled page offering both the existing Table view and a new Trail view (via `pages-event-trail` + `hostPanel()`) with expandable payload details, correlation tracking, and shared filters.

**Architecture:** The audit page uses `tabs()` DSL to toggle between the existing DSL table ("Table") and `pages-event-trail` via `hostPanel()` ("Trail"). Four targeted upstream fixes harden `pages-event-trail` in casehub-pages. TypeScript discriminated union types model the `BridgeMessage` wire format. Correlation siblings are fetched server-side on expand via `/api/bridge/audit?correlationId=X` and cached per page session.

**Tech Stack:** TypeScript (Lit, pages-ui DSL, pages-event-trail), Java (Quarkus REST), vitest (unit tests)

## Global Constraints

- `pages-event-trail` upstream fixes target `casehub-pages` repo at `/Users/mdproctor/claude/casehub/pages`
- IoT webapp TypeScript lives at `webapp/src/main/webapp/src/`
- All DSL imports from `@casehubio/pages-ui`, runtime from `@casehubio/pages-runtime`
- `pages-event-trail` component from `@casehubio/pages-ui-components/event-trail`
- TypeScript types must match actual Jackson serialization output (verified during spec review)
- `BridgeAuditEventType` has 8 values; `AGENT_CONNECTED`/`AGENT_DISCONNECTED` have null `message`
- `HeartbeatPayload` is NOT in the union — no `HEARTBEAT` audit event type exists
- `TypedRow` cells use `row.text()`, `row.date()`, `row.cell()` — NOT `row.get()`
- REST endpoint `GET /api/bridge/audit` accepts: `eventType`, `deviceId`, `correlationId`, `from`, `to`, `offset`, `limit` (max 500)

---

## Batch 1: Upstream pages-event-trail fixes

> **Repo:** `/Users/mdproctor/claude/casehub/pages` (casehub-pages)
> After this batch: `pages-event-trail` supports `getRowDetail`/`getRowKey`/`columnRenderers` via `configure()`, `recordsPath` for wrapped responses, `csvExport` passthrough, and raw entry pass-through for detail renderers.

### Task 1: Harden pages-event-trail for hostPanel consumers

**Files:**
- Modify: `packages/pages-ui-components/src/event-trail/pages-event-trail.ts`
- Modify: `packages/pages-ui-components/src/event-trail/pages-event-trail.test.ts`

**Interfaces:**
- Consumes: `TypedRow`, `ColumnId`, `ColumnRenderer` from `@casehubio/pages-data` / `@casehubio/pages-table`
- Produces: Updated `PagesEventTrail` component with:
  - `configure(props)` handling `getRowDetail`, `getRowKey`, `columnRenderers`, `recordsPath`, `csvExport`
  - `recordsPath` property for extracting arrays from wrapped responses
  - `csvExport` property passed through to internal `pages-table`
  - `_rawEntryByRow: WeakMap<TypedRow, unknown>` mapping rows to raw entries
  - `getRowDetail` signature: `(row: TypedRow, rawEntry?: unknown) => TemplateResult | undefined`

- [ ] **Step 1: Write failing test for configure() passthrough**

In `pages-event-trail.test.ts`, add a test that verifies `getRowDetail`, `getRowKey`, and `columnRenderers` survive `configure()`:

```typescript
it('configure() passes through getRowDetail, getRowKey, and columnRenderers', () => {
  const el = new PagesEventTrail();
  const detail = (row: TypedRow) => html`<div>detail</div>`;
  const key = (row: TypedRow) => 'key';
  const renderers = new Map();

  el.configure({ getRowDetail: detail, getRowKey: key, columnRenderers: renderers });

  expect(el.getRowDetail).toBe(detail);
  expect(el.getRowKey).toBe(key);
  expect(el.columnRenderers).toBe(renderers);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/pages-ui-components test -- --run`
Expected: FAIL — `getRowDetail` is undefined after configure

- [ ] **Step 3: Implement configure() passthrough**

In `pages-event-trail.ts`, add property passthrough in the `configure()` method, after the existing property handling and before `super.configure(props)`:

```typescript
if (props.getRowDetail !== undefined) this.getRowDetail = props.getRowDetail as (row: TypedRow, rawEntry?: unknown) => TemplateResult | undefined;
if (props.getRowKey !== undefined) this.getRowKey = props.getRowKey as (row: TypedRow) => string;
if (props.columnRenderers !== undefined) this.columnRenderers = props.columnRenderers as ReadonlyMap<ColumnId, ColumnRenderer>;
```

Also update the `getRowDetail` property type declaration:

```typescript
@property({ type: Object }) getRowDetail?: (row: TypedRow, rawEntry?: unknown) => TemplateResult | undefined;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/pages-ui-components test -- --run`
Expected: PASS

- [ ] **Step 5: Write failing test for recordsPath**

```typescript
it('createSourceFactory() extracts records via recordsPath', async () => {
  const el = new PagesEventTrail();
  el.recordsPath = 'records';
  el.columnDefs = [{ id: 'name', label: 'Name', getValue: (r: any) => r.name }];

  const mockResponse = { records: [{ name: 'Alice' }, { name: 'Bob' }], totalCount: 2 };
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(mockResponse),
  });

  el.endpoint = '/api/test';
  el.syncEndpoint();
  await vi.waitFor(() => expect(el['_rawEntries']).toHaveLength(2));
  expect(el['_rawEntries'][0]).toEqual({ name: 'Alice' });
});
```

- [ ] **Step 6: Run test to verify it fails**

Expected: FAIL — `_rawEntries` contains the wrapper object, not the extracted array

- [ ] **Step 7: Implement recordsPath**

Add property declaration:

```typescript
@property({ type: String }) recordsPath?: string;
```

Add to `configure()`:

```typescript
if (props.recordsPath !== undefined) this.recordsPath = props.recordsPath as string;
```

In `createSourceFactory()`, modify the `.then((entries: unknown[])` handler to extract via recordsPath:

```typescript
.then((body: unknown) => {
  if (signal.aborted) return;
  const entries = (this.recordsPath
    ? (body as Record<string, unknown>)[this.recordsPath]
    : body) as unknown[];
  this._rawEntries = entries;
  // ... rest unchanged
})
```

- [ ] **Step 8: Run test to verify it passes**

Expected: PASS

- [ ] **Step 9: Write failing test for csvExport**

```typescript
it('csvExport is passed through to internal pages-table', async () => {
  const el = new PagesEventTrail();
  el.csvExport = true;
  el.columnDefs = [{ id: 'x', label: 'X', getValue: () => '' }];
  el.data = [{ x: 'v' }];
  el.requestUpdate();
  await el.updateComplete;

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table?.csvExport).toBe(true);
});
```

- [ ] **Step 10: Run test to verify it fails**

Expected: FAIL — `csvExport` property not set on `pages-table`

- [ ] **Step 11: Implement csvExport passthrough**

Add property:

```typescript
@property({ type: Boolean }) csvExport = false;
```

Add to `configure()`:

```typescript
if (props.csvExport !== undefined) this.csvExport = props.csvExport as boolean;
```

In `render()`, add to the `<pages-table>` element:

```typescript
?csvExport=${this.csvExport}
```

- [ ] **Step 12: Run test to verify it passes**

Expected: PASS

- [ ] **Step 13: Write failing test for raw entry pass-through**

```typescript
it('getRowDetail receives raw entry as second parameter', async () => {
  const rawEntries: unknown[] = [];
  const el = new PagesEventTrail();
  el.columnDefs = [{ id: 'name', label: 'Name', getValue: (r: any) => r.name }];
  el.data = [{ name: 'Alice', nested: { deep: true } }];
  el.getRowDetail = (_row: TypedRow, rawEntry?: unknown) => {
    rawEntries.push(rawEntry);
    return html`<div>detail</div>`;
  };
  el.requestUpdate();
  await el.updateComplete;

  // Trigger detail expansion
  const table = el.shadowRoot!.querySelector('pages-table') as any;
  const firstRow = table?.dataSet?.rows[0];
  if (firstRow && table?.getRowDetail) {
    table.getRowDetail(firstRow);
  }

  expect(rawEntries).toHaveLength(1);
  expect(rawEntries[0]).toEqual({ name: 'Alice', nested: { deep: true } });
});
```

- [ ] **Step 14: Run test to verify it fails**

Expected: FAIL — `rawEntry` is undefined

- [ ] **Step 15: Implement raw entry WeakMap**

Add state property:

```typescript
@state() private _rawEntryByRow = new WeakMap<TypedRow, unknown>();
```

In `_applyFilters()`, after building `_filteredDataSet`, build the WeakMap:

```typescript
this._rawEntryByRow = new WeakMap();
const activeDataSet = this._filteredDataSet ?? this.dataSet;
if (activeDataSet) {
  for (let i = 0; i < activeDataSet.rows.length; i++) {
    const row = activeDataSet.rows[i];
    if (row) this._rawEntryByRow.set(row, filtered[i]);
  }
}
```

In `render()`, wrap `getRowDetail` before passing to `pages-table`:

```typescript
const wrappedDetail = this.getRowDetail
  ? (row: TypedRow) => this.getRowDetail!(row, this._rawEntryByRow.get(row))
  : undefined;
```

Then on the `pages-table` element:

```typescript
.getRowDetail=${wrappedDetail}
```

(replacing the existing `.getRowDetail=${this.getRowDetail}`)

- [ ] **Step 16: Run test to verify it passes**

Expected: PASS

- [ ] **Step 17: Run full test suite**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/pages-ui-components test -- --run`
Expected: All tests pass

- [ ] **Step 18: Commit upstream changes**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-ui-components/src/event-trail/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: harden pages-event-trail for hostPanel consumers

Add configure() passthrough for getRowDetail, getRowKey, columnRenderers.
Add recordsPath for wrapped endpoint responses.
Add csvExport passthrough to internal pages-table.
Add raw entry pass-through via WeakMap for detail renderers needing
complex nested objects that CellValue conversion would destroy.

Refs casehubio/iot#103"
```

- [ ] **Step 19: Build and install SNAPSHOT**

```bash
yarn --cwd /Users/mdproctor/claude/casehub/pages build
yarn --cwd /Users/mdproctor/claude/casehub/pages workspace @casehubio/pages-ui-components pack
```

Then update the IoT webapp's dependency to pick up the changes (the exact mechanism depends on the casehub-packages sync workflow — follow the existing pattern in the IoT repo).

---

## Batch 2: TypeScript types and payload renderers

> **Repo:** `/Users/mdproctor/claude/casehub/iot` (IoT webapp)
> After this batch: TypeScript type model exists for bridge audit wire format, and all 8 event type renderers produce correct Lit templates from fixture data. No page wiring yet.

### Task 2: TypeScript type model and vitest setup

**Files:**
- Create: `webapp/src/main/webapp/src/types/bridge-audit.ts`
- Modify: `webapp/src/main/webapp/package.json` (add vitest dev dependency)
- Create: `webapp/src/main/webapp/vitest.config.ts`

**Interfaces:**
- Consumes: Nothing
- Produces:
  - `DeviceEntity` interface with `@deviceType` discriminator and index signature
  - `StateChangeEvent` interface with `before`/`after`/`changedCapabilities`
  - `BridgeMessagePayload` discriminated union (6 variants: StateChange, ReplayedStateChange, StateSnapshot, ProviderStatus, Command, CommandResponse)
  - `BridgeAuditEventType` string union (8 values)
  - `AuditRecord` interface

- [ ] **Step 1: Add vitest to the IoT webapp**

Add vitest as a dev dependency in `webapp/src/main/webapp/package.json`:

```json
"devDependencies": {
  "vitest": "^3.2.1"
}
```

Add test script:

```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 2: Create vitest config**

Create `webapp/src/main/webapp/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/**/*.test.ts'],
    environment: 'jsdom',
  },
});
```

- [ ] **Step 3: Install dependencies**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp install`

- [ ] **Step 4: Create TypeScript type model**

Create `webapp/src/main/webapp/src/types/bridge-audit.ts` with all types from the spec's §Detail rendering — TypeScript type model section. This includes:

```typescript
export interface DeviceEntity {
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

export interface StateChangeEvent {
  before: DeviceEntity | null;
  after: DeviceEntity;
  changedCapabilities: string[];
  occurredAt: string;
  providerId: string;
}

export interface StateChangePayload {
  "@type": "STATE_CHANGE";
  tenancyId: string;
  timestamp: string;
  event: StateChangeEvent;
}

export interface ReplayedStateChangePayload {
  "@type": "REPLAYED_STATE_CHANGE";
  tenancyId: string;
  timestamp: string;
  event: StateChangeEvent;
}

export interface StateSnapshotPayload {
  "@type": "STATE_SNAPSHOT";
  tenancyId: string;
  timestamp: string;
  devices: DeviceEntity[];
}

export interface ProviderStatusPayload {
  "@type": "PROVIDER_STATUS";
  tenancyId: string;
  timestamp: string;
  status: {
    providerId: string;
    previousStatus: "CONNECTED" | "CONNECTING" | "DISCONNECTED";
    currentStatus: "CONNECTED" | "CONNECTING" | "DISCONNECTED";
  };
}

export interface CommandPayload {
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

export interface CommandResponsePayload {
  "@type": "COMMAND_RESULT";
  tenancyId: string;
  timestamp: string;
  correlationId: string;
  result: "SENT" | "FAILED" | "TIMEOUT";
}

export type BridgeMessagePayload =
  | StateChangePayload
  | ReplayedStateChangePayload
  | StateSnapshotPayload
  | ProviderStatusPayload
  | CommandPayload
  | CommandResponsePayload;

export type BridgeAuditEventType =
  | "STATE_CHANGE"
  | "REPLAYED_STATE_CHANGE"
  | "STATE_SNAPSHOT"
  | "PROVIDER_STATUS_CHANGE"
  | "COMMAND_SENT"
  | "COMMAND_RESPONSE"
  | "AGENT_CONNECTED"
  | "AGENT_DISCONNECTED";

export interface AuditRecord {
  eventType: BridgeAuditEventType;
  deviceId: string | null;
  correlationId: string | null;
  payload: BridgeMessagePayload | null;
  occurredAt: string;
}
```

- [ ] **Step 5: Verify types compile**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp tsc --noEmit`
Expected: No errors

- [ ] **Step 6: Write exhaustiveness test**

Create `webapp/src/main/webapp/src/types/bridge-audit.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import type { BridgeMessagePayload, BridgeAuditEventType, AuditRecord } from './bridge-audit';

describe('BridgeMessagePayload exhaustiveness', () => {
  function assertExhaustive(payload: BridgeMessagePayload): string {
    switch (payload["@type"]) {
      case "STATE_CHANGE": return "state-change";
      case "REPLAYED_STATE_CHANGE": return "replayed";
      case "STATE_SNAPSHOT": return "snapshot";
      case "PROVIDER_STATUS": return "provider";
      case "COMMAND": return "command";
      case "COMMAND_RESULT": return "response";
      default: {
        const _exhaustive: never = payload;
        return _exhaustive;
      }
    }
  }

  it('covers all 6 payload variants', () => {
    const variants: BridgeMessagePayload[] = [
      { "@type": "STATE_CHANGE", tenancyId: "t", timestamp: "t", event: { before: null, after: { "@deviceType": "Light", deviceId: "d1", deviceClass: "light", label: "L", available: true, lastUpdated: "", tenancyId: "t", providerId: "p", location: null }, changedCapabilities: [], occurredAt: "", providerId: "p" } },
      { "@type": "REPLAYED_STATE_CHANGE", tenancyId: "t", timestamp: "t", event: { before: null, after: { "@deviceType": "Light", deviceId: "d1", deviceClass: "light", label: "L", available: true, lastUpdated: "", tenancyId: "t", providerId: "p", location: null }, changedCapabilities: [], occurredAt: "", providerId: "p" } },
      { "@type": "STATE_SNAPSHOT", tenancyId: "t", timestamp: "t", devices: [] },
      { "@type": "PROVIDER_STATUS", tenancyId: "t", timestamp: "t", status: { providerId: "p", previousStatus: "DISCONNECTED", currentStatus: "CONNECTED" } },
      { "@type": "COMMAND", tenancyId: "t", timestamp: "t", correlationId: "c", command: { targetDeviceId: "d", action: "turnOn", parameters: {}, dispatchedBy: "user", correlationId: "c" } },
      { "@type": "COMMAND_RESULT", tenancyId: "t", timestamp: "t", correlationId: "c", result: "SENT" },
    ];

    const results = variants.map(assertExhaustive);
    expect(results).toEqual(["state-change", "replayed", "snapshot", "provider", "command", "response"]);
  });

  it('AuditRecord allows null payload for lifecycle events', () => {
    const record: AuditRecord = {
      eventType: "AGENT_CONNECTED",
      deviceId: null,
      correlationId: null,
      payload: null,
      occurredAt: "2026-09-25T10:00:00Z",
    };
    expect(record.payload).toBeNull();
  });
});
```

- [ ] **Step 7: Run test**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/types/ webapp/src/main/webapp/package.json webapp/src/main/webapp/vitest.config.ts
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#103): add TypeScript type model for bridge audit wire format

Discriminated union types mirror Java BridgeMessage sealed interface.
Exhaustiveness test verifies all 6 payload variants compile.
Adds vitest for IoT webapp TypeScript unit testing.

Refs #103"
```

### Task 3: Payload renderers

**Files:**
- Create: `webapp/src/main/webapp/src/renderers/audit-detail.ts`
- Create: `webapp/src/main/webapp/src/renderers/audit-detail.test.ts`

**Interfaces:**
- Consumes: `AuditRecord`, `BridgeAuditEventType`, `BridgeMessagePayload` from `../types/bridge-audit`
- Produces:
  - `renderAuditDetail(row: TypedRow, rawEntry?: unknown): TemplateResult | undefined` — main entry point for `getRowDetail`
  - `renderPayload(eventType: BridgeAuditEventType, payload: BridgeMessagePayload | null): TemplateResult` — per-type dispatch
  - `auditRowKey(row: TypedRow): string` — row key derivation

- [ ] **Step 1: Write failing test for STATE_CHANGE renderer**

Create `webapp/src/main/webapp/src/renderers/audit-detail.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { renderPayload } from './audit-detail';
import type { StateChangePayload } from '../types/bridge-audit';

describe('renderPayload', () => {
  it('renders STATE_CHANGE with device label and capability diff', () => {
    const payload: StateChangePayload = {
      "@type": "STATE_CHANGE",
      tenancyId: "t1",
      timestamp: "2026-09-25T10:00:00Z",
      event: {
        before: {
          "@deviceType": "Light", deviceId: "light-1", deviceClass: "light",
          label: "Kitchen Light", available: true, lastUpdated: "", tenancyId: "t1",
          providerId: "ha", location: null, brightness: 50,
        },
        after: {
          "@deviceType": "Light", deviceId: "light-1", deviceClass: "light",
          label: "Kitchen Light", available: true, lastUpdated: "", tenancyId: "t1",
          providerId: "ha", location: null, brightness: 100,
        },
        changedCapabilities: ["brightness"],
        occurredAt: "2026-09-25T10:00:00Z",
        providerId: "ha",
      },
    };

    const result = renderPayload("STATE_CHANGE", payload);
    expect(result).toBeDefined();
    // Verify the template contains expected content
    const values = (result as any).values;
    expect(JSON.stringify(values)).toContain("Kitchen Light");
    expect(JSON.stringify(values)).toContain("brightness");
  });

  it('renders AGENT_CONNECTED with null payload', () => {
    const result = renderPayload("AGENT_CONNECTED", null);
    expect(result).toBeDefined();
    const values = (result as any).values;
    expect(JSON.stringify(values)).toContain("connected");
  });

  it('renders AGENT_DISCONNECTED with null payload', () => {
    const result = renderPayload("AGENT_DISCONNECTED", null);
    expect(result).toBeDefined();
    const values = (result as any).values;
    expect(JSON.stringify(values)).toContain("disconnected");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: FAIL — `renderPayload` not found

- [ ] **Step 3: Implement renderPayload**

Create `webapp/src/main/webapp/src/renderers/audit-detail.ts`:

```typescript
import { html, nothing, type TemplateResult } from 'lit';
import type {
  AuditRecord, BridgeAuditEventType, BridgeMessagePayload,
  StateChangePayload, ReplayedStateChangePayload, StateSnapshotPayload,
  ProviderStatusPayload, CommandPayload, CommandResponsePayload,
  StateChangeEvent,
} from '../types/bridge-audit';
import type { TypedRow } from '@casehubio/pages-data';
import type { ColumnId } from '@casehubio/pages-data';

export function renderPayload(
  eventType: BridgeAuditEventType,
  payload: BridgeMessagePayload | null,
): TemplateResult {
  if (payload === null) {
    return renderLifecycleEvent(eventType);
  }

  switch (payload["@type"]) {
    case "STATE_CHANGE":
      return renderStateChange(payload.event, false);
    case "REPLAYED_STATE_CHANGE":
      return renderStateChange(payload.event, true);
    case "STATE_SNAPSHOT":
      return renderStateSnapshot(payload);
    case "PROVIDER_STATUS":
      return renderProviderStatus(payload);
    case "COMMAND":
      return renderCommand(payload);
    case "COMMAND_RESULT":
      return renderCommandResponse(payload);
    default: {
      const _exhaustive: never = payload;
      return html`<span>Unknown payload type</span>`;
    }
  }
}

function renderLifecycleEvent(eventType: BridgeAuditEventType): TemplateResult {
  const label = eventType === "AGENT_CONNECTED"
    ? "Bridge agent connected"
    : "Bridge agent disconnected";
  const badge = eventType === "AGENT_CONNECTED" ? "connected" : "disconnected";
  return html`
    <div class="lifecycle-event">
      <span class="badge badge-${badge}">${label}</span>
    </div>
  `;
}

function renderStateChange(event: StateChangeEvent, replayed: boolean): TemplateResult {
  const device = event.after;
  return html`
    <div class="state-change-detail">
      ${replayed ? html`<span class="badge badge-replayed">Replayed</span>` : nothing}
      <div class="device-header">
        <strong>${device.label}</strong> <code>${device.deviceId}</code>
        <span class="device-type">${device["@deviceType"]}</span>
      </div>
      ${event.changedCapabilities.length > 0 ? html`
        <table class="capability-diff">
          <thead><tr><th>Capability</th><th>Before</th><th>After</th></tr></thead>
          <tbody>
            ${event.changedCapabilities.map(cap => html`
              <tr>
                <td>${cap}</td>
                <td>${event.before ? String(event.before[cap] ?? '—') : '—'}</td>
                <td>${String(device[cap] ?? '—')}</td>
              </tr>
            `)}
          </tbody>
        </table>
      ` : html`<p>No capability changes recorded</p>`}
    </div>
  `;
}

function renderStateSnapshot(payload: StateSnapshotPayload): TemplateResult {
  return html`
    <div class="state-snapshot-detail">
      <p><strong>${payload.devices.length}</strong> device(s) in snapshot</p>
      <table class="snapshot-devices">
        <thead><tr><th>Type</th><th>Device ID</th><th>Label</th><th>Available</th></tr></thead>
        <tbody>
          ${payload.devices.map(d => html`
            <tr>
              <td>${d["@deviceType"]}</td>
              <td><code>${d.deviceId}</code></td>
              <td>${d.label}</td>
              <td>${d.available ? '✓' : '✗'}</td>
            </tr>
          `)}
        </tbody>
      </table>
    </div>
  `;
}

function renderProviderStatus(payload: ProviderStatusPayload): TemplateResult {
  return html`
    <div class="provider-status-detail">
      <p>Provider <strong>${payload.status.providerId}</strong></p>
      <p>
        <span class="badge badge-${payload.status.previousStatus.toLowerCase()}">${payload.status.previousStatus}</span>
        → <span class="badge badge-${payload.status.currentStatus.toLowerCase()}">${payload.status.currentStatus}</span>
      </p>
    </div>
  `;
}

function renderCommand(payload: CommandPayload): TemplateResult {
  const cmd = payload.command;
  return html`
    <div class="command-detail">
      <p>Target: <code>${cmd.targetDeviceId}</code> — Action: <strong>${cmd.action}</strong></p>
      <p>Dispatched by: ${cmd.dispatchedBy}</p>
      ${Object.keys(cmd.parameters).length > 0 ? html`
        <table class="command-params">
          <thead><tr><th>Parameter</th><th>Value</th></tr></thead>
          <tbody>
            ${Object.entries(cmd.parameters).map(([k, v]) => html`
              <tr><td>${k}</td><td>${JSON.stringify(v)}</td></tr>
            `)}
          </tbody>
        </table>
      ` : nothing}
    </div>
  `;
}

function renderCommandResponse(payload: CommandResponsePayload): TemplateResult {
  const badgeClass = payload.result === "SENT" ? "success"
    : payload.result === "FAILED" ? "error" : "warning";
  return html`
    <div class="command-response-detail">
      <span class="badge badge-${badgeClass}">${payload.result}</span>
    </div>
  `;
}

export function auditRowKey(row: TypedRow): string {
  const at = row.date("occurredAt" as ColumnId).toISOString();
  const type = row.text("eventType" as ColumnId);
  const devCell = row.cell("deviceId" as ColumnId);
  const dev = devCell.type === "NULL" ? "system" : (devCell as { value: string }).value;
  return `${at}-${type}-${dev}`;
}

export function renderAuditDetail(row: TypedRow, rawEntry?: unknown): TemplateResult | undefined {
  const record = rawEntry as AuditRecord | undefined;
  if (!record) return undefined;

  return html`
    <div class="audit-detail">
      ${renderPayload(record.eventType, record.payload)}
      ${record.correlationId ? html`<div class="correlation-placeholder" data-correlation-id="${record.correlationId}"></div>` : nothing}
    </div>
  `;
}
```

Note: The correlation section is a placeholder (`correlation-placeholder` div) that will be wired in Batch 4.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: PASS

- [ ] **Step 5: Add tests for remaining payload types**

Add to `audit-detail.test.ts`:

```typescript
it('renders STATE_SNAPSHOT with device count', () => {
  const result = renderPayload("STATE_SNAPSHOT", {
    "@type": "STATE_SNAPSHOT", tenancyId: "t", timestamp: "t",
    devices: [
      { "@deviceType": "Light", deviceId: "l1", deviceClass: "light", label: "L1", available: true, lastUpdated: "", tenancyId: "t", providerId: "p", location: null },
      { "@deviceType": "Switch", deviceId: "s1", deviceClass: "switch", label: "S1", available: false, lastUpdated: "", tenancyId: "t", providerId: "p", location: null },
    ],
  });
  const values = JSON.stringify((result as any).values);
  expect(values).toContain("2");
});

it('renders PROVIDER_STATUS_CHANGE with transition', () => {
  const result = renderPayload("PROVIDER_STATUS_CHANGE", {
    "@type": "PROVIDER_STATUS", tenancyId: "t", timestamp: "t",
    status: { providerId: "homeassistant", previousStatus: "DISCONNECTED", currentStatus: "CONNECTED" },
  });
  const values = JSON.stringify((result as any).values);
  expect(values).toContain("homeassistant");
  expect(values).toContain("DISCONNECTED");
  expect(values).toContain("CONNECTED");
});

it('renders COMMAND_SENT with target and action', () => {
  const result = renderPayload("COMMAND_SENT", {
    "@type": "COMMAND", tenancyId: "t", timestamp: "t", correlationId: "c1",
    command: { targetDeviceId: "light-1", action: "turnOn", parameters: { brightness: 100 }, dispatchedBy: "admin", correlationId: "c1" },
  });
  const values = JSON.stringify((result as any).values);
  expect(values).toContain("light-1");
  expect(values).toContain("turnOn");
});

it('renders COMMAND_RESPONSE with result badge', () => {
  const result = renderPayload("COMMAND_RESPONSE", {
    "@type": "COMMAND_RESULT", tenancyId: "t", timestamp: "t", correlationId: "c1", result: "FAILED",
  });
  const values = JSON.stringify((result as any).values);
  expect(values).toContain("FAILED");
  expect(values).toContain("error");
});

it('renders REPLAYED_STATE_CHANGE with badge', () => {
  const result = renderPayload("REPLAYED_STATE_CHANGE", {
    "@type": "REPLAYED_STATE_CHANGE", tenancyId: "t", timestamp: "t",
    event: {
      before: null,
      after: { "@deviceType": "Light", deviceId: "d1", deviceClass: "light", label: "L", available: true, lastUpdated: "", tenancyId: "t", providerId: "p", location: null },
      changedCapabilities: [],
      occurredAt: "", providerId: "p",
    },
  });
  const values = JSON.stringify((result as any).values);
  expect(values).toContain("Replayed");
});
```

- [ ] **Step 6: Run all tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/renderers/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#103): add tailored audit detail renderers for all BridgeMessage variants

Renderers branch on eventType first (handles null-message lifecycle events),
then on @type for message-bearing events. StateChange shows capability diff
table. Command shows target/action/params. Correlation placeholder wired
in Batch 4.

Refs #103"
```

---

## Batch 3: Tab-toggled audit page

> **Repo:** `/Users/mdproctor/claude/casehub/iot` (IoT webapp)
> After this batch: The audit page has Table and Trail tabs. Trail view uses pages-event-trail with expandable payload details. Page-level filters drive both views via a filter adapter. Correlation placeholder is visible but not yet functional.

### Task 4: Component registration + audit page refactor + filter adapter

**Files:**
- Modify: `webapp/src/main/webapp/src/app.ts`
- Modify: `webapp/src/main/webapp/src/pages/audit.ts`
- Create: `webapp/src/main/webapp/src/components/audit-filter-adapter.ts`

**Interfaces:**
- Consumes:
  - `renderAuditDetail` from `../renderers/audit-detail` (Task 3)
  - `auditRowKey` from `../renderers/audit-detail` (Task 3)
  - `registerPanel` from `@casehubio/pages-runtime`
  - `tabs`, `hostPanel` from `@casehubio/pages-ui`
- Produces:
  - Tab-toggled audit page with shared filter state
  - `AuditFilterAdapter` custom element that listens for DSL filter group changes and updates the Trail panel's endpoint URL

- [ ] **Step 1: Add component registration to app.ts**

In `webapp/src/main/webapp/src/app.ts`, add imports and registration:

After the existing imports add:

```typescript
import '@casehubio/pages-ui-components/event-trail';
import { registerPanel } from "@casehubio/pages-runtime";
import "./components/audit-filter-adapter";
```

After the dataset declarations, before the app construction, add:

```typescript
registerPanel("event-trail", "pages-event-trail");
```

- [ ] **Step 2: Create the filter adapter**

Create `webapp/src/main/webapp/src/components/audit-filter-adapter.ts`. This is a headless custom element (~25 lines) that bridges the DSL filter group to the Trail panel's endpoint:

```typescript
import { LitElement } from 'lit';

const BASE_ENDPOINT = '/api/bridge/audit?limit=500';

export class AuditFilterAdapter extends LitElement {
  private _trailPanel: HTMLElement | null = null;

  override connectedCallback(): void {
    super.connectedCallback();
    this.style.display = 'none';
    this._findTrailPanel();
    this.addEventListener('filter-group-change', this._onFilterChange as EventListener);
    document.addEventListener('tab-change', this._onTabChange as EventListener);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.removeEventListener('filter-group-change', this._onFilterChange as EventListener);
    document.removeEventListener('tab-change', this._onTabChange as EventListener);
  }

  private _findTrailPanel(): void {
    this._trailPanel = document.querySelector('pages-event-trail');
  }

  private _onFilterChange = (e: CustomEvent): void => {
    this._syncTrail(e.detail);
  };

  private _onTabChange = (): void => {
    if (!this._trailPanel) this._findTrailPanel();
    if (this._trailPanel) this._syncTrail(this._getCurrentFilters());
  };

  private _getCurrentFilters(): Record<string, string> {
    const params: Record<string, string> = {};
    const selectors = this.closest('[data-filter-group="audit"]')
      ?.querySelectorAll('[data-filter-field]') ?? [];
    for (const sel of selectors) {
      const field = sel.getAttribute('data-filter-field');
      const value = (sel as any).value;
      if (field && value) params[field] = value;
    }
    return params;
  }

  private _syncTrail(filters: Record<string, string>): void {
    if (!this._trailPanel) this._findTrailPanel();
    if (!this._trailPanel) return;

    const url = new URL(BASE_ENDPOINT, globalThis.location.origin);
    for (const [key, value] of Object.entries(filters)) {
      if (value) url.searchParams.set(key, value);
    }
    (this._trailPanel as any).endpoint = url.pathname + url.search;
    (this._trailPanel as any).syncEndpoint?.();
  }
}

if (!customElements.get('audit-filter-adapter')) {
  customElements.define('audit-filter-adapter', AuditFilterAdapter);
}
```

**Note:** The exact filter event and DOM structure depends on how pages-ui exposes filter group changes. The adapter's event listening and DOM queries may need adjustment during implementation based on the actual pages-runtime event model. The spec identifies this as a ~20 line adapter with two responsibilities: (1) sync on filter change, (2) sync on tab activation.

- [ ] **Step 3: Refactor audit.ts to tab-toggled layout**

Replace the contents of `webapp/src/main/webapp/src/pages/audit.ts` with:

```typescript
import { page, rows, tabs, panel, table, columns, selector,
         datePicker, lookup, groupBy, col, sortBy, hostPanel } from "@casehubio/pages-ui";
import { renderAuditDetail, auditRowKey } from "../renderers/audit-detail";

export function auditPage() {
  return page("Audit",
    rows(
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
            { id: "eventType", label: "Event", getValue: (r: any) => r.eventType },
            { id: "deviceId", label: "Device", getValue: (r: any) => r.deviceId },
            { id: "correlationId", label: "Correlation", getValue: (r: any) => r.correlationId },
            { id: "occurredAt", label: "Time", getValue: (r: any) => r.occurredAt },
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

- [ ] **Step 4: Verify TypeScript compilation**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp tsc --noEmit`
Expected: No errors

- [ ] **Step 5: Build the webapp**

Run: `mvn --batch-mode -pl webapp -am install -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 6: Start the webapp and verify the audit page**

Start the dev server and navigate to the Audit page. Verify:
- Both "Table" and "Trail" tabs are visible
- Table tab shows the existing table (same as before)
- Trail tab shows a pages-event-trail with data loaded from `/api/bridge/audit?limit=500`
- Expanding a row in the Trail shows the tailored detail pane
- CSV export works in both tabs
- Applying a filter (e.g. selecting an eventType) in the Table tab, then switching to Trail shows filtered data

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/app.ts webapp/src/main/webapp/src/pages/audit.ts webapp/src/main/webapp/src/components/audit-filter-adapter.ts
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#103): tab-toggled audit page with Table and Trail views

Trail view uses pages-event-trail via hostPanel with expandable
detail rendering for all BridgeMessage variants. Page-level filters
above tabs drive both views via filter adapter. CSV export enabled
in both tabs.

Refs #103"
```

---

## Batch 4: Correlation display

> **Repo:** `/Users/mdproctor/claude/casehub/iot` (IoT webapp)
> After this batch: Expanding a row with a correlationId shows the payload detail plus a "Related Events" section with all correlated siblings fetched server-side. Cache prevents re-fetching.

### Task 5: Correlation fetch, cache, and render

**Files:**
- Create: `webapp/src/main/webapp/src/renderers/audit-correlation.ts`
- Create: `webapp/src/main/webapp/src/renderers/audit-correlation.test.ts`
- Modify: `webapp/src/main/webapp/src/renderers/audit-detail.ts`

**Interfaces:**
- Consumes: `AuditRecord` from `../types/bridge-audit`
- Produces:
  - `renderCorrelation(correlationId: string): TemplateResult` — returns Lit template using `until()` directive
  - `fetchCorrelation(correlationId: string): Promise<AuditRecord[]>` — cached fetch
  - `clearCorrelationCache(): void` — for testing

- [ ] **Step 1: Write failing test for fetchCorrelation**

Create `webapp/src/main/webapp/src/renderers/audit-correlation.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { fetchCorrelation, clearCorrelationCache } from './audit-correlation';

describe('fetchCorrelation', () => {
  beforeEach(() => {
    clearCorrelationCache();
    vi.restoreAllMocks();
  });

  it('fetches related events by correlationId', async () => {
    const mockRecords = [
      { eventType: "COMMAND_SENT", deviceId: "d1", correlationId: "c1", payload: null, occurredAt: "2026-09-25T10:00:00Z" },
      { eventType: "COMMAND_RESPONSE", deviceId: "d1", correlationId: "c1", payload: null, occurredAt: "2026-09-25T10:00:01Z" },
    ];
    globalThis.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ records: mockRecords }),
    });

    const result = await fetchCorrelation("c1");
    expect(result).toEqual(mockRecords);
    expect(globalThis.fetch).toHaveBeenCalledWith("/api/bridge/audit?correlationId=c1");
  });

  it('caches results and does not re-fetch', async () => {
    globalThis.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ records: [] }),
    });

    await fetchCorrelation("c2");
    await fetchCorrelation("c2");
    expect(globalThis.fetch).toHaveBeenCalledTimes(1);
  });

  it('evicts cache entry on fetch failure and retries', async () => {
    let callCount = 0;
    globalThis.fetch = vi.fn().mockImplementation(() => {
      callCount++;
      if (callCount === 1) return Promise.resolve({ ok: false, status: 500 });
      return Promise.resolve({ ok: true, json: () => Promise.resolve({ records: [{ eventType: "COMMAND_SENT" }] }) });
    });

    await expect(fetchCorrelation("c3")).rejects.toThrow();
    const result = await fetchCorrelation("c3");
    expect(result).toHaveLength(1);
    expect(globalThis.fetch).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: FAIL — module not found

- [ ] **Step 3: Implement correlation fetch with cache**

Create `webapp/src/main/webapp/src/renderers/audit-correlation.ts`:

```typescript
import { html, type TemplateResult } from 'lit';
import { until } from 'lit/directives/until.js';
import type { AuditRecord, BridgeAuditEventType } from '../types/bridge-audit';

const correlationCache = new Map<string, Promise<AuditRecord[]>>();

export function clearCorrelationCache(): void {
  correlationCache.clear();
}

export function fetchCorrelation(correlationId: string): Promise<AuditRecord[]> {
  if (!correlationCache.has(correlationId)) {
    correlationCache.set(correlationId,
      fetch(`/api/bridge/audit?correlationId=${correlationId}`)
        .then(r => {
          if (!r.ok) throw new Error(`HTTP ${r.status}`);
          return r.json();
        })
        .then(body => body.records as AuditRecord[])
        .catch(err => {
          correlationCache.delete(correlationId);
          throw err;
        })
    );
  }
  return correlationCache.get(correlationId)!;
}

const EVENT_TYPE_LABELS: Record<BridgeAuditEventType, string> = {
  STATE_CHANGE: "State Change",
  REPLAYED_STATE_CHANGE: "Replayed",
  STATE_SNAPSHOT: "Snapshot",
  PROVIDER_STATUS_CHANGE: "Provider",
  COMMAND_SENT: "Command",
  COMMAND_RESPONSE: "Response",
  AGENT_CONNECTED: "Connected",
  AGENT_DISCONNECTED: "Disconnected",
};

function renderRelatedRow(record: AuditRecord, isCurrent: boolean): TemplateResult {
  return html`
    <tr class="${isCurrent ? 'current-event' : ''}">
      <td>${new Date(record.occurredAt).toLocaleTimeString()}</td>
      <td><span class="badge badge-${record.eventType.toLowerCase()}">${EVENT_TYPE_LABELS[record.eventType]}</span></td>
      <td>${record.deviceId ?? '—'}</td>
    </tr>
  `;
}

export function renderCorrelation(correlationId: string, currentOccurredAt?: string): TemplateResult {
  const promise = fetchCorrelation(correlationId)
    .then(events => {
      if (events.length <= 1) return html``;
      return html`
        <div class="related-events">
          <h4>Related Events</h4>
          <table class="related-events-table">
            <thead><tr><th>Time</th><th>Event</th><th>Device</th></tr></thead>
            <tbody>
              ${events.map(e => renderRelatedRow(e, e.occurredAt === currentOccurredAt))}
            </tbody>
          </table>
        </div>
      `;
    })
    .catch(() => html`<span class="error">Failed to load related events</span>`);

  return html`${until(promise, html`<span class="loading">Loading related events...</span>`)}`;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: All PASS

- [ ] **Step 5: Wire correlation into renderAuditDetail**

In `webapp/src/main/webapp/src/renderers/audit-detail.ts`, replace the correlation placeholder with the actual render call.

Replace the import section to add:

```typescript
import { renderCorrelation } from './audit-correlation';
```

Replace the `renderAuditDetail` function body:

```typescript
export function renderAuditDetail(row: TypedRow, rawEntry?: unknown): TemplateResult | undefined {
  const record = rawEntry as AuditRecord | undefined;
  if (!record) return undefined;

  return html`
    <div class="audit-detail">
      ${renderPayload(record.eventType, record.payload)}
      ${record.correlationId
        ? renderCorrelation(record.correlationId, record.occurredAt)
        : nothing}
    </div>
  `;
}
```

- [ ] **Step 6: Run all tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/iot/webapp/src/main/webapp test -- --run`
Expected: All PASS

- [ ] **Step 7: Build and verify**

Run: `mvn --batch-mode -pl webapp -am install -DskipTests`
Expected: BUILD SUCCESS

Start the dev server and verify:
- Expanding a row with a correlationId shows the payload detail + "Related Events" section
- "Loading related events..." appears briefly, then the correlated events render
- Collapsing and re-expanding the same row does NOT re-fetch (cached)
- Expanding a row WITHOUT a correlationId shows only the payload detail

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/renderers/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#103): add correlation display with server-side fetch and caching

Expanding a row with a correlationId fetches related events via
/api/bridge/audit?correlationId=X. Results are cached per correlationId
until page unmount. Failed fetches are evicted from cache for retry.
Uses Lit until() directive for async rendering.

Refs #103"
```

---

## References

- [2026-09-25-bridge-audit-trail-viewer-design.md] — design spec this plan implements
- [decisions.md] — 5 decisions (all revised per light review)
- `webapp/src/main/webapp/src/pages/audit.ts` — current DSL table (34 lines)
- `webapp/src/main/webapp/src/app.ts` — application shell with dataset declarations
- `packages/pages-ui-components/src/event-trail/pages-event-trail.ts` — upstream component (casehub-pages)
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeMessage.java` — 7 sealed variants
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditEventType.java` — 8 event types
- `api/src/main/java/io/casehub/iot/api/bridge/BridgeAuditQuery.java` — correlationId query support
- `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTOperationsApi.java:94-118` — getBridgeAudit REST endpoint
- `webapp-api/src/main/java/io/casehub/iot/webapp/view/AuditTrailView.java` — response shape
- GitHub #103 — focal issue
- Spec review R1-02, R1-03, R1-05, R1-06, R1-07, R1-09, R1-10, R1-13, R1-16, R2-02, R2-03, R2-04 — verified findings incorporated
