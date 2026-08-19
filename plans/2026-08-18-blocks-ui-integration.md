# blocks-ui Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #95 — Replace manual webapp UI with blocks-ui components
**Issue group:** #95

**Goal:** Replace manual pages-ui DSL construction in KPI rows, work items page, and device detail with blocks-ui web components via `hostPanel()`.

**Architecture:** Three phases — KPI rows first (validates the hostPanel integration lifecycle), work items second (split layout with blocks-ui detail), device detail third (requires blocks-ui configure() addition). Java REST endpoints provide server-computed KPI metrics. TypeScript pages embed blocks-ui components via pages-ui `hostPanel()` DSL function.

**Tech Stack:** Java 21, Quarkus 3.32.2, TypeScript 5.4, Lit 3, pages-ui DSL, blocks-ui web components, Vite

## Global Constraints

- `hostPanel(typeName, panelProps)` is the integration mechanism — never use `html()` for components that need complex properties
- `ConfigurablePanel.configure(props)` is called before `connectedCallback()` — components must store state without triggering render
- blocks-ui components consumed via `.casehub-packages` portal resolutions (Maven SNAPSHOT `casehub-blocks-ui-npm`)
- `KpiMetric` Java record mirrors blocks-ui `MetricDefinition` by convention, not coupling — fields evolve independently
- Audit page stays unchanged (out of scope — see spec D1)

---

## Batch 1: KPI Row Integration (validates hostPanel lifecycle)

### Task 1: KPI REST Endpoint + hostPanel Validation

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/KpiMetric.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/KpiResource.java`
- Create: `webapp/src/test/java/io/casehub/iot/webapp/app/rest/KpiResourceTest.java`
- Modify: `webapp/src/main/webapp/src/app.ts`
- Modify: `webapp/src/main/webapp/src/pages/devices.ts`
- Modify: `webapp/src/main/webapp/src/pages/health.ts`
- Modify: `webapp/src/main/webapp/package.json`
- Delete: `webapp/src/main/webapp/src/components/device-kpi-row.ts`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/rest/KpiResourceTest.java`

**Interfaces:**
- Produces: `KpiMetric(key, value, label, unit, status)` — record returned by REST endpoints
- Produces: `GET /api/devices/kpi` → `List<KpiMetric>`
- Produces: `GET /api/health/kpi` → `List<KpiMetric>`
- Consumes: `DeviceRegistry.findAll()`, `Instance<DeviceProvider>`, `BridgeConnectionRegistry`

- [ ] **Step 1: Create KpiMetric record in webapp-api**

```java
// webapp-api/src/main/java/io/casehub/iot/webapp/rest/KpiMetric.java
package io.casehub.iot.webapp.rest;

public record KpiMetric(
    String key,
    Object value,
    String label,
    String unit,
    String status
) {}
```

Use `ide_create_file` to create the file. This record mirrors blocks-ui `MetricDefinition` by convention.

- [ ] **Step 2: Write failing test for device KPI endpoint**

```java
// webapp/src/test/java/io/casehub/iot/webapp/app/rest/KpiResourceTest.java
package io.casehub.iot.webapp.app.rest;

import io.casehub.iot.api.*;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.webapp.rest.KpiMetric;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class KpiResourceTest {

    private KpiResource resource;
    private TestDeviceRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new TestDeviceRegistry();
        resource = new KpiResource();
        resource.deviceRegistry = registry;
    }

    @Test
    void deviceKpiReturnsFourMetrics() {
        registry.devices = List.of(
            SwitchDevice.builder().deviceId("sw1").tenancyId("t1")
                .providerId("openhab").label("Switch").available(true)
                .on(true).build(),
            LightDevice.builder().deviceId("lt1").tenancyId("t1")
                .providerId("ha").label("Light").available(false)
                .on(false).build()
        );

        List<KpiMetric> metrics = resource.deviceKpi("t1");

        assertThat(metrics).hasSize(4);
        assertThat(metrics).extracting(KpiMetric::key)
            .containsExactlyInAnyOrder("total-devices", "online", "providers", "active-alerts");

        var total = metrics.stream().filter(m -> "total-devices".equals(m.key())).findFirst().orElseThrow();
        assertThat(total.value()).isEqualTo(2L);

        var online = metrics.stream().filter(m -> "online".equals(m.key())).findFirst().orElseThrow();
        assertThat(online.value()).isEqualTo(1L);
        assertThat(online.status()).isEqualTo("warning");

        var providers = metrics.stream().filter(m -> "providers".equals(m.key())).findFirst().orElseThrow();
        assertThat(providers.value()).isEqualTo(2L);
    }

    @Test
    void deviceKpiOnlineStatusNormalAbove50Percent() {
        registry.devices = List.of(
            SwitchDevice.builder().deviceId("sw1").tenancyId("t1")
                .providerId("ha").label("S1").available(true).on(true).build(),
            SwitchDevice.builder().deviceId("sw2").tenancyId("t1")
                .providerId("ha").label("S2").available(true).on(true).build()
        );

        List<KpiMetric> metrics = resource.deviceKpi("t1");

        var online = metrics.stream().filter(m -> "online".equals(m.key())).findFirst().orElseThrow();
        assertThat(online.status()).isEqualTo("normal");
    }

    static class TestDeviceRegistry implements DeviceRegistry {
        List<DeviceEntity> devices = List.of();

        @Override public java.util.Optional<DeviceEntity> findById(String id) { return java.util.Optional.empty(); }
        @Override public <T extends DeviceEntity> List<T> findByClass(Class<T> c) { return List.of(); }
        @Override public List<DeviceEntity> findByTenancyId(String t) {
            return devices.stream().filter(d -> t.equals(d.tenancyId())).toList();
        }
        @Override public List<DeviceEntity> findAll() { return devices; }
        @Override public void refresh() {}
        @Override public void refresh(String p) {}
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl webapp -Dtest=KpiResourceTest -DfailIfNoTests=false -q`
Expected: compilation failure — `KpiResource` class does not exist

- [ ] **Step 4: Implement KpiResource**

```java
// webapp/src/main/java/io/casehub/iot/webapp/app/rest/KpiResource.java
package io.casehub.iot.webapp.app.rest;

import io.casehub.iot.api.IoTRoles;
import io.casehub.iot.api.ProviderStatus;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.bridge.server.BridgeConnectionRegistry;
import io.casehub.iot.webapp.rest.KpiMetric;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

import java.util.List;

@Path("/api")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class KpiResource {

    @Inject DeviceRegistry deviceRegistry;
    @Inject Instance<DeviceProvider> providers;
    @Inject CurrentPrincipal principal;
    @Inject BridgeConnectionRegistry connectionRegistry;

    @GET
    @Path("/devices/kpi")
    @RolesAllowed(IoTRoles.VIEWER)
    public List<KpiMetric> deviceKpi() {
        return deviceKpi(principal.tenancyId());
    }

    List<KpiMetric> deviceKpi(String tenancyId) {
        var devices = deviceRegistry.findAll().stream()
                .filter(d -> d.tenancyId().equals(tenancyId))
                .toList();

        long total = devices.size();
        long online = devices.stream().filter(d -> d.available()).count();
        long providerCount = devices.stream().map(d -> d.providerId()).distinct().count();

        String onlineStatus = total > 0 && online * 2 < total ? "warning" : "normal";

        return List.of(
            new KpiMetric("total-devices", total, "Total Devices", null, "normal"),
            new KpiMetric("online", online, "Online", null, onlineStatus),
            new KpiMetric("providers", providerCount, "Providers", null, "normal"),
            new KpiMetric("active-alerts", 0L, "Active Alerts", null, "normal")
        );
    }

    @GET
    @Path("/health/kpi")
    @RolesAllowed(IoTRoles.VIEWER)
    public List<KpiMetric> healthKpi() {
        long connectedProviders = providers.stream()
                .filter(p -> p.status() == ProviderStatus.CONNECTED)
                .count();
        long bridgeConnections = connectionRegistry.connectedTenancies().size();

        return List.of(
            new KpiMetric("connected-providers", connectedProviders, "Connected Providers", null, "normal"),
            new KpiMetric("bridge-connections", bridgeConnections, "Bridge Connections", null, "normal"),
            new KpiMetric("active-situations", 0L, "Active Situations", null, "normal"),
            new KpiMetric("open-cases", 0L, "Open Cases", null, "normal")
        );
    }
}
```

Use `ide_create_file` to create the file.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl webapp -Dtest=KpiResourceTest -q`
Expected: PASS — all assertions green

- [ ] **Step 6: Add blocks-ui component imports to app.ts**

Modify `webapp/src/main/webapp/src/app.ts` — add these imports after the existing page imports:

```typescript
// blocks-ui component registration (custom element side-effect imports)
import "@casehubio/blocks-ui/kpi-metric-row";
```

Add portal resolution to `package.json`:
```json
"@casehubio/blocks-ui/kpi-metric-row": "portal:./.casehub-packages/kpi-metric-row"
```

Use `ide_replace_text_in_file` for the app.ts change.

- [ ] **Step 7: Rewrite devices.ts KPI row**

Replace the `deviceKpiRow("devices")` call with `hostPanel`:

```typescript
import { page, rows, columns, panel, selector, title, table, hostPanel, html, lookup, groupBy, col, filterBy, sortBy } from "@casehubio/pages-ui";
import { dataSetId, columnId } from "@casehubio/pages-data/dist/dataset/types.js";
import { deviceTable } from "../components/device-table";

export function devicesPage() {
  return page("Devices",
    rows(
      // KPI row — blocks-ui component via hostPanel
      hostPanel("blocks-kpi-metric-row", {
        endpoint: "/api/devices/kpi",
        columns: 4,
        density: "comfortable",
      }),

      // Device class filter (unchanged)
      columns([2, 10],
        // ... rest unchanged
```

Remove the `import { deviceKpiRow }` line.

- [ ] **Step 8: Rewrite health.ts KPI row**

Replace the manual `columns([3,3,3,3], [panel(...)])` with `hostPanel`:

```typescript
import { page, rows, panel, table, hostPanel, lookup } from "@casehubio/pages-ui";

export function healthPage() {
  return page("Health",
    rows(
      // KPI row — blocks-ui component via hostPanel
      hostPanel("blocks-kpi-metric-row", {
        endpoint: "/api/health/kpi",
        columns: 4,
        density: "comfortable",
      }),

      // Provider status table (unchanged)
      panel("Provider Status", table({
        title: "Providers",
        sortable: true,
        lookup: lookup("providers"),
        refresh: { interval: 10000 },
      })),
    ),
  );
}
```

- [ ] **Step 9: Delete device-kpi-row.ts**

Use `ide_refactor_safe_delete` on `webapp/src/main/webapp/src/components/device-kpi-row.ts`.

- [ ] **Step 10: Build and verify**

Run: `mvn install -pl webapp-api,webapp -DskipTests -q`
Expected: compilation passes

Then start dev mode and verify visually:
Run: `mvn quarkus:dev -pl webapp -Dquarkus.http.port=8081`
Expected: devices page shows blocks-ui KPI cards with 4 metrics from `/api/devices/kpi`

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp-api/src/main/java/io/casehub/iot/webapp/rest/KpiMetric.java webapp/src/main/java/io/casehub/iot/webapp/app/rest/KpiResource.java webapp/src/test/java/io/casehub/iot/webapp/app/rest/KpiResourceTest.java webapp/src/main/webapp/src/app.ts webapp/src/main/webapp/src/pages/devices.ts webapp/src/main/webapp/src/pages/health.ts webapp/src/main/webapp/package.json
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#95): KPI rows via hostPanel + blocks-kpi-metric-row

Add KpiResource REST endpoints (/api/devices/kpi, /api/health/kpi)
returning MetricDefinition[] for blocks-kpi-metric-row consumption.
Replace manual DSL metric construction with hostPanel() integration.
Delete device-kpi-row.ts component.

Refs #95"
```

---

## Batch 2: Work Items Split Layout

### Task 2: Work Items Master-Detail with blocks-work-item-detail

**Files:**
- Modify: `webapp/src/main/webapp/src/app.ts`
- Modify: `webapp/src/main/webapp/src/pages/workitems.ts`
- Modify: `webapp/src/main/webapp/package.json`

**Interfaces:**
- Consumes: `hostPanel("blocks-work-item-detail", props)` — blocks-ui work item lifecycle component
- Consumes: `split("horizontal", [...])` — pages-ui split layout function
- Consumes: `dataset("workitems", "/api/workitems")` — existing dataset declaration in app.ts

- [ ] **Step 1: Add blocks-work-item-detail import to app.ts**

```typescript
import "@casehubio/blocks-ui/work-item-detail";
```

Add portal resolution to `package.json`:
```json
"@casehubio/blocks-ui/work-item-detail": "portal:./.casehub-packages/work-item-detail"
```

- [ ] **Step 2: Rewrite workitems.ts with split layout**

```typescript
import { page, rows, panel, table, split, hostPanel, selector, columns, lookup, groupBy, col, sortBy } from "@casehubio/pages-ui";

export function workItemsPage() {
  return page("Work Items",
    rows(
      // Status filter
      columns([2, 10],
        [selector({
          title: "Status",
          filter: { enabled: true },
          lookup: lookup("workitems", groupBy("status", col("status"))),
          subtype: "labels",
        })],
      ),

      // Master-detail split: table on left, work-item-detail on right
      split("horizontal",
        [
          panel("Tasks", table({
            title: "WorkItems",
            sortable: true,
            pageSize: 20,
            filter: { listening: true },
            lookup: lookup("workitems", sortBy("createdAt", "DESCENDING")),
            refresh: { interval: 15000 },
          })),
          hostPanel("blocks-work-item-detail", {
            endpoint: "/api",
          }),
        ],
        { ratio: [0.4, 0.6], minSizes: [320, 400] },
      ),
    ),
  );
}
```

This replaces the inline HTML buttons (Claim/Approve/Reject) with the full blocks-ui work item lifecycle component (claim/start/complete/reject/suspend/resume/cancel/release/delegate/escalate).

Note: The selection wiring between the table and the detail panel depends on how `blocks-work-item-detail` subscribes to selection events. It listens for `WorkItemEventTopics.SELECTED` events — the table's row click needs to emit this event. This may require a custom event handler or the pages-ui `dataScope` mechanism. Verify during integration testing.

- [ ] **Step 3: Build and verify**

Run: `mvn install -pl webapp -DskipTests -q`
Expected: compilation passes

Start dev mode and verify:
- Split pane renders with table on left, detail on right
- Table shows work items from `/api/workitems`
- Clicking a work item row populates the detail panel
- Lifecycle actions (claim, complete, etc.) are visible and functional

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/app.ts webapp/src/main/webapp/src/pages/workitems.ts webapp/src/main/webapp/package.json
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#95): work items page with split layout + blocks-work-item-detail

Replace manual table + inline HTML buttons with split() layout
composing pages-ui table and hostPanel(blocks-work-item-detail).
Full work item lifecycle management via blocks-ui component.

Refs #95"
```

---

## Batch 3: Device Detail Tabbed Pane

### Task 3: Add configure() to blocks-detail-pane (cross-repo)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/blocks-ui/components/detail-pane/src/detail-pane.ts`
- Modify: `/Users/mdproctor/claude/casehub/blocks-ui/components/detail-pane/src/types.ts`

**Interfaces:**
- Produces: `DetailPane.configure(props: Record<string, unknown>): void` — satisfies `ConfigurablePanel`

- [ ] **Step 1: Read the current DetailPane class to find the insertion point**

Use `ide_file_structure` on the blocks-ui repo (open via `ide_open_project` if needed).

- [ ] **Step 2: Add configure() method to DetailPane**

Add after the `emptyMessage` property declaration:

```typescript
configure(props: Record<string, unknown>): void {
  if (props.tabs !== undefined) this.tabs = props.tabs as TabDefinition[];
  if (props.selectionTopic !== undefined) this.selectionTopic = props.selectionTopic as string;
  if (props.emptyMessage !== undefined) this.emptyMessage = props.emptyMessage as string;
}
```

Use `ide_insert_member` with member name `configure` on the `DetailPane` class.

- [ ] **Step 3: Build blocks-ui and install SNAPSHOT**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui
yarn build
mvn install -q
```

This publishes the updated `casehub-blocks-ui-npm` SNAPSHOT to `~/.m2`.

- [ ] **Step 4: Commit in blocks-ui**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/detail-pane/src/detail-pane.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add configure() to DetailPane for hostPanel integration

Implements ConfigurablePanel interface so pages-ui hostPanel() can
pass tabs, selectionTopic, and emptyMessage as panelProps.

Refs casehubio/iot#95"
```

### Task 4: Device Detail with blocks-detail-pane

**Files:**
- Modify: `webapp/src/main/webapp/src/app.ts`
- Modify: `webapp/src/main/webapp/src/pages/devices.ts`
- Modify: `webapp/src/main/webapp/package.json`

**Interfaces:**
- Consumes: `hostPanel("blocks-detail-pane", { tabs, selectionTopic })` — tabbed detail panel
- Consumes: `TabDefinition` from `@casehubio/blocks-ui/detail-pane`

- [ ] **Step 1: Unpack updated blocks-ui SNAPSHOT**

```bash
mvn dependency:unpack -pl webapp -Dartifact=io.casehub:casehub-blocks-ui-npm:0.1-SNAPSHOT -DoutputDirectory=webapp/src/main/webapp/.casehub-packages
```

Or re-run the existing initialize phase:
```bash
mvn initialize -pl webapp
```

- [ ] **Step 2: Add blocks-detail-pane import to app.ts**

```typescript
import "@casehubio/blocks-ui/detail-pane";
```

Add portal resolution to `package.json`:
```json
"@casehubio/blocks-ui/detail-pane": "portal:./.casehub-packages/detail-pane"
```

- [ ] **Step 3: Rewrite device detail sub-page in devices.ts**

Replace the inline metric cards and HTML action buttons with a `hostPanel("blocks-detail-pane")`:

```typescript
// Device detail sub-page
page("Device Detail",
  rows(
    title("Device Details"),

    hostPanel("blocks-detail-pane", {
      selectionTopic: "device",
      tabs: [
        {
          id: "state",
          label: "State",
          tagName: "div",  // placeholder — renders device capabilities
          order: 0,
        },
        {
          id: "history",
          label: "History",
          tagName: "div",  // placeholder — renders state change timeline
          order: 1,
        },
        {
          id: "actions",
          label: "Actions",
          tagName: "div",  // placeholder — renders command dispatch buttons
          order: 2,
        },
      ],
    }),
  ),
  {
    dataScope: { dataset: dataSetId("devices"), idColumn: columnId("deviceId") },
  },
),
```

Note: The tab content elements (`tagName: "div"`) are placeholders. For a production implementation, each tab would use a custom element that receives the selected device item via the `item` property (set by `DetailPane._getOrCreateTabElement`). Creating these tab content elements is part of the implementation work — they are simple Lit elements that render device state, history table, or action buttons respectively.

- [ ] **Step 4: Build and verify**

Run: `mvn install -pl webapp -DskipTests -q`
Expected: compilation passes

Start dev mode and verify:
- Device detail sub-page shows tabbed interface
- Three tabs: State, History, Actions
- Tab switching works via click and arrow keys
- Selected device populates the detail pane

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add webapp/src/main/webapp/src/app.ts webapp/src/main/webapp/src/pages/devices.ts webapp/src/main/webapp/package.json
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#95): device detail with blocks-detail-pane tabbed view

Replace inline metric cards and HTML action buttons with
hostPanel(blocks-detail-pane) providing tabbed State/History/Actions
views with keyboard navigation.

Refs #95"
```

---

## References

- [2026-08-18-blocks-ui-integration-design.md] — design spec this plan implements
- [webapp/src/main/webapp/src/pages/devices.ts] — current devices page
- [webapp/src/main/webapp/src/pages/health.ts] — current health page
- [webapp/src/main/webapp/src/pages/workitems.ts] — current work items page
- [webapp/src/main/webapp/src/components/device-kpi-row.ts] — component to delete
- [blocks-ui/components/kpi-metric-row/src/kpi-metric-row.ts:260] — configure() method
- [blocks-ui/components/work-item-detail/src/work-item-detail.ts:443] — configure() method
- [blocks-ui/components/detail-pane/src/detail-pane.ts] — needs configure() added
- [pages-ui/dist/dsl/builders.d.ts:72] — hostPanel() signature
- [pages-component/dist/model/hosting.d.ts:16] — ConfigurablePanel interface
- [GE-20260712-7250c5] — DataSourceMixin extraction pipeline issues
- [GE-20260717-19540a] — esbuild TC39 decorator fix
- [GE-20260717-c99f50] — TypedRow property access pattern
- [GitHub #95] — focal issue
