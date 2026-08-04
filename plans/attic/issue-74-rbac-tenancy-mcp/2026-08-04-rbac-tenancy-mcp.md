# RBAC, Tenancy Filtering, and Principal Propagation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #74 — RBAC, tenancy filtering, and principal propagation for MCP tools
**Issue group:** #74

**Goal:** Add role-based access control, tenant-scoped data filtering, and caller identity propagation to the four MCP tools.

**Architecture:** Inline `@RolesAllowed` annotations on tool methods. `McpIdentityContext` bean encapsulates principal resolution with graceful fallback for hosts without OIDC. Tenancy filtering uses existing `DeviceRegistry.findByTenancyId()` and a new `findById(String, String)` default method. `DeviceStateHistoryProvider.findHistory()` gains a `tenancyId` parameter for data-layer enforcement.

**Tech Stack:** Quarkus CDI, Jakarta Security (`@RolesAllowed`), `casehub-platform-api` (`CurrentPrincipal`), Arc container introspection

**Spec:** `specs/issue-74-rbac-tenancy-mcp/2026-08-02-rbac-tenancy-mcp-design.md`

## Global Constraints

- MCP module is a library — must not break bridge (no OIDC, no CurrentPrincipal)
- Single tenancy property: `casehub.iot.tenancy-id`
- `casehub-iot-api` is public API — breaking changes require major version bump (pre-release, so acceptable)
- `DeviceEntity.tenancyId()` is never null (`Objects.requireNonNull` in constructor)
- Use `Instance<CurrentPrincipal>` (not direct `@Inject`) — deferred resolution for optional beans
- Three-guard pattern for scope safety: `isResolvable()` → `Arc.container() != null` → `requestContext().isActive()`

---

### Task 1: IoTRoles Constants and Webapp @RolesAllowed Migration

**Files:**
- Create: `api/src/main/java/io/casehub/iot/api/IoTRoles.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceResource.java` — migrate 3x `@RolesAllowed` string literals to constants; fix `dispatch()` `dispatchedBy` from `tenancyId()` to `actorId()`; simplify `filterByTenancy()` dead null check
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java` — migrate 6x `@RolesAllowed` string literals to constants
- Modify: `ARC42STORIES.MD` — refine §2 authorization-agnostic constraint

**Interfaces:**
- Produces: `IoTRoles.VIEWER = "iot-viewer"`, `IoTRoles.OPERATOR = "iot-operator"`, `IoTRoles.ADMIN = "iot-admin"` — used by Task 4

- [ ] **Step 1: Create IoTRoles.java**

Use `ide_create_file`:

```java
package io.casehub.iot.api;

public final class IoTRoles {
    public static final String VIEWER = "iot-viewer";
    public static final String OPERATOR = "iot-operator";
    public static final String ADMIN = "iot-admin";
    private IoTRoles() {}
}
```

File: `api/src/main/java/io/casehub/iot/api/IoTRoles.java`

- [ ] **Step 2: Migrate DeviceResource @RolesAllowed to constants**

Use `ide_replace_text_in_file` three times on `webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceResource.java`:

1. Replace `@RolesAllowed("iot-viewer")` → `@RolesAllowed(IoTRoles.VIEWER)` (2 occurrences — `list()` at line 67, `get()` at line 99)
2. Replace `@RolesAllowed("iot-operator")` → `@RolesAllowed(IoTRoles.OPERATOR)` (1 occurrence — `dispatch()` at line 131)
3. Add import: `import io.casehub.iot.api.IoTRoles;`

- [ ] **Step 3: Fix DeviceResource.dispatch() dispatchedBy bug**

In `DeviceResource.java` line 152, replace:
```java
principal.tenancyId(), // dispatchedBy
```
with:
```java
principal.actorId(), // dispatchedBy
```

- [ ] **Step 4: Simplify DeviceResource.filterByTenancy()**

Replace the `filterByTenancy` method body. `DeviceEntity.tenancyId()` is never null — the null check is dead code:

```java
private boolean filterByTenancy(String deviceTenancyId) {
    return deviceTenancyId.equals(principal.tenancyId());
}
```

- [ ] **Step 5: Migrate SituationResource @RolesAllowed to constants**

Use `ide_replace_text_in_file` on `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java`:

1. Replace `@RolesAllowed("iot-viewer")` → `@RolesAllowed(IoTRoles.VIEWER)` (3 occurrences — lines 132, 284, 339)
2. Replace `@RolesAllowed("iot-admin")` → `@RolesAllowed(IoTRoles.ADMIN)` (3 occurrences — lines 164, 214, 267)
3. Add import: `import io.casehub.iot.api.IoTRoles;`

- [ ] **Step 6: Update ARC42STORIES.MD §2**

Find the constraint line "Authorization-agnostic: `iot-api` does not enforce authorization." and refine to:
"Authorization-agnostic: `iot-api` does not enforce authorization; it may define role name constants used by consuming modules for annotation-based access control."

- [ ] **Step 7: Build and verify**

Run: `mvn --batch-mode install -pl api,webapp -am`
Expected: BUILD SUCCESS. All existing tests pass — this is a pure string-to-constant migration.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add api/src/main/java/io/casehub/iot/api/IoTRoles.java webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceResource.java webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#74): IoTRoles constants and webapp @RolesAllowed migration"
```

---

### Task 2: API Tenancy Extensions

**Files:**
- Modify: `api/src/main/java/io/casehub/iot/api/spi/DeviceRegistry.java` — add `findById(String, String)` default method
- Modify: `api/src/main/java/io/casehub/iot/api/spi/DeviceStateHistoryProvider.java` — add `tenancyId` param to `findHistory()`
- Modify: `api/src/main/java/io/casehub/iot/api/IoTCommandAuditEvent.java` — add `tenancyId` record component
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/JpaDeviceStateHistoryProvider.java` — add `AND h.tenancyId = :tenancyId`
- Modify: `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java` — update `fireAuditEvent()` to pass tenancyId (temporary null until Task 4 wires McpIdentityContext)
- Test: `api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java` — test findById with tenancy

**Interfaces:**
- Produces: `DeviceRegistry.findById(String deviceId, String tenancyId)` — used by Task 4
- Produces: `DeviceStateHistoryProvider.findHistory(String deviceId, String tenancyId, Instant from, Instant to, int limit)` — used by Task 4
- Produces: `IoTCommandAuditEvent(... String tenancyId, Instant timestamp)` — used by Task 4

- [ ] **Step 1: Write failing test for DeviceRegistry.findById tenancy filtering**

Add to `api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java` using `ide_insert_member`:

```java
@Test
void findByIdWithTenancyReturnsMatchingDevice() {
    registry.refresh();
    var result = registry.findById("light.living_room", "test-tenant");
    assertThat(result).isPresent();
    assertThat(result.get().tenancyId()).isEqualTo("test-tenant");
}

@Test
void findByIdWithTenancyRejectsWrongTenant() {
    registry.refresh();
    var result = registry.findById("light.living_room", "other-tenant");
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl api -Dtest=CdiDeviceRegistryTest#findByIdWithTenancy*`
Expected: FAIL — `findById(String, String)` does not exist

- [ ] **Step 3: Add findById default method to DeviceRegistry**

Use `ide_insert_member` on `api/src/main/java/io/casehub/iot/api/spi/DeviceRegistry.java`, after `findById(String)`:

```java
default Optional<DeviceEntity> findById(String deviceId, String tenancyId) {
    return findById(deviceId).filter(d -> d.tenancyId().equals(tenancyId));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl api -Dtest=CdiDeviceRegistryTest#findByIdWithTenancy*`
Expected: PASS

- [ ] **Step 5: Update DeviceStateHistoryProvider.findHistory() signature**

Use `ide_change_signature` on `api/src/main/java/io/casehub/iot/api/spi/DeviceStateHistoryProvider.java` line 18, `findHistory` method — add `tenancyId` parameter after `deviceId`:

New signature: `List<HistoryEntry> findHistory(String deviceId, String tenancyId, Instant from, Instant to, int limit)`

- [ ] **Step 6: Update JpaDeviceStateHistoryProvider**

Use `ide_replace_member` on `webapp/src/main/java/io/casehub/iot/webapp/app/persistence/JpaDeviceStateHistoryProvider.java`, method `findHistory`:

```java
@Override
public List<HistoryEntry> findHistory(String deviceId, String tenancyId, Instant from, Instant to, int limit) {
    var query = em.createQuery(
            """
            SELECT h FROM IoTDeviceStateHistoryEntity h
            WHERE h.deviceId = :deviceId
              AND h.tenancyId = :tenancyId
              AND (:from IS NULL OR h.occurredAt >= :from)
              AND (:to IS NULL OR h.occurredAt <= :to)
            ORDER BY h.occurredAt DESC
            """,
            IoTDeviceStateHistoryEntity.class
    );

    query.setParameter("deviceId", deviceId);
    query.setParameter("tenancyId", tenancyId);
    query.setParameter("from", from);
    query.setParameter("to", to);
    query.setMaxResults(limit);

    return query.getResultList().stream()
            .map(h -> new HistoryEntry(
                    h.getDeviceId(),
                    h.getDeviceClass(),
                    h.getStateSnapshot(),
                    Arrays.asList(h.getChangedCapabilities()),
                    h.getOccurredAt()
            ))
            .toList();
}
```

- [ ] **Step 7: Add tenancyId to IoTCommandAuditEvent**

Use `ide_edit_member` on `api/src/main/java/io/casehub/iot/api/IoTCommandAuditEvent.java`, member `IoTCommandAuditEvent`:

```java
public record IoTCommandAuditEvent(
    String deviceId,
    String action,
    Map<String, Object> parameters,
    CommandResult result,
    String dispatchedBy,
    String correlationId,
    String providerId,
    String tenancyId,
    Instant timestamp
) {}
```

- [ ] **Step 8: Fix IoTDeviceMcpTool.fireAuditEvent() compilation**

The record constructor now requires `tenancyId`. Update `fireAuditEvent` in `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java` to pass a placeholder (will be replaced by `identityContext.tenancyId()` in Task 4):

Use `ide_replace_member` on `fireAuditEvent`:

```java
private void fireAuditEvent(DeviceCommand command, CommandResult result, String providerId) {
    auditEvents.fireAsync(new IoTCommandAuditEvent(
            command.targetDeviceId(),
            command.action(),
            command.parameters(),
            result,
            command.dispatchedBy(),
            command.correlationId(),
            providerId,
            "pending-identity-context",
            Instant.now()));
}
```

- [ ] **Step 9: Fix IoTDeviceMcpTool.getHistory() compilation**

The `findHistory` call now needs `tenancyId`. Temporarily pass `"pending-identity-context"` (replaced in Task 4):

In `getHistory()` method, change:
```java
var entries = historyProvider.findHistory(deviceId, fromInstant, toInstant, effectiveLimit);
```
to:
```java
var entries = historyProvider.findHistory(deviceId, "pending-identity-context", fromInstant, toInstant, effectiveLimit);
```

- [ ] **Step 10: Fix DeviceResource.history() compilation**

`DeviceResource` calls `findHistory` — update to pass `principal.tenancyId()`:

Find the `findHistory` call in DeviceResource and add `principal.tenancyId()` as the second argument.

- [ ] **Step 11: Fix IoTDeviceMcpToolTest compilation**

All `findHistory` mock setups need the new `tenancyId` parameter. Update `setUp()` and history tests to pass `any()` or `eq("default-tenant")` for the tenancyId parameter in mock verifications and stub setups.

- [ ] **Step 12: Build and verify**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all existing tests adapted.

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add api/ mcp/ webapp/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#74): API tenancy extensions — findById tenancy, findHistory tenancyId, audit tenancyId"
```

---

### Task 3: McpIdentityContext

**Files:**
- Create: `mcp/src/main/java/io/casehub/iot/mcp/McpIdentityContext.java`
- Create: `mcp/src/test/java/io/casehub/iot/mcp/McpIdentityContextTest.java`

**Interfaces:**
- Produces: `McpIdentityContext.tenancyId()` → `String`, `McpIdentityContext.actorId()` → `String` — used by Task 4

- [ ] **Step 1: Write failing tests for McpIdentityContext**

Create `mcp/src/test/java/io/casehub/iot/mcp/McpIdentityContextTest.java` using `ide_create_file`:

```java
package io.casehub.iot.mcp;

import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class McpIdentityContextTest {

    private static final String CONFIG_TENANCY = "config-tenant-id";

    @Test
    void tenancyIdReturnsPrincipalTenancy() {
        var principal = stubPrincipal("tenant-from-principal", "user-123");
        var ctx = withPrincipal(principal);
        assertThat(ctx.tenancyId()).isEqualTo("tenant-from-principal");
    }

    @Test
    void tenancyIdFallsBackToConfig() {
        var ctx = withoutPrincipal();
        assertThat(ctx.tenancyId()).isEqualTo(CONFIG_TENANCY);
    }

    @Test
    void actorIdReturnsPrincipalActor() {
        var principal = stubPrincipal("some-tenant", "user-456");
        var ctx = withPrincipal(principal);
        assertThat(ctx.actorId()).isEqualTo("user-456");
    }

    @Test
    void actorIdFallsBackToMcpAgent() {
        var ctx = withoutPrincipal();
        assertThat(ctx.actorId()).isEqualTo("mcp-agent");
    }

    @SuppressWarnings("unchecked")
    private McpIdentityContext withPrincipal(CurrentPrincipal principal) {
        Instance<CurrentPrincipal> instance = mock(Instance.class);
        when(instance.isResolvable()).thenReturn(true);
        when(instance.get()).thenReturn(principal);
        return new McpIdentityContext(instance, CONFIG_TENANCY) {
            @Override
            boolean isPrincipalAvailable() {
                return true;
            }
        };
    }

    @SuppressWarnings("unchecked")
    private McpIdentityContext withoutPrincipal() {
        Instance<CurrentPrincipal> instance = mock(Instance.class);
        when(instance.isResolvable()).thenReturn(false);
        return new McpIdentityContext(instance, CONFIG_TENANCY);
    }

    private CurrentPrincipal stubPrincipal(String tenancyId, String actorId) {
        return new CurrentPrincipal() {
            @Override public String actorId() { return actorId; }
            @Override public Set<String> groups() { return Set.of(); }
            @Override public String tenancyId() { return tenancyId; }
            @Override public boolean isCrossTenantAdmin() { return false; }
        };
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl mcp -Dtest=McpIdentityContextTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement McpIdentityContext**

Create `mcp/src/main/java/io/casehub/iot/mcp/McpIdentityContext.java` using `ide_create_file`:

```java
package io.casehub.iot.mcp;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.arc.Arc;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class McpIdentityContext {

    private final Instance<CurrentPrincipal> currentPrincipal;
    private final String configTenancyId;

    @Inject
    public McpIdentityContext(Instance<CurrentPrincipal> currentPrincipal,
                              @ConfigProperty(name = "casehub.iot.tenancy-id")
                              String configTenancyId) {
        this.currentPrincipal = currentPrincipal;
        this.configTenancyId = configTenancyId;
    }

    boolean isPrincipalAvailable() {
        return currentPrincipal.isResolvable()
                && Arc.container() != null
                && Arc.container().requestContext().isActive();
    }

    public String tenancyId() {
        if (isPrincipalAvailable()) {
            return currentPrincipal.get().tenancyId();
        }
        return configTenancyId;
    }

    public String actorId() {
        if (isPrincipalAvailable()) {
            return currentPrincipal.get().actorId();
        }
        return "mcp-agent";
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl mcp -Dtest=McpIdentityContextTest`
Expected: PASS — all 4 tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add mcp/src/main/java/io/casehub/iot/mcp/McpIdentityContext.java mcp/src/test/java/io/casehub/iot/mcp/McpIdentityContextTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#74): McpIdentityContext — principal resolution with graceful fallback"
```

---

### Task 4: IoTDeviceMcpTool RBAC, Tenancy Filtering, and Identity Propagation

**Files:**
- Modify: `mcp/src/main/java/io/casehub/iot/mcp/IoTDeviceMcpTool.java` — add `@RolesAllowed`, inject `McpIdentityContext`, wire tenancy filtering and actorId
- Modify: `mcp/src/test/java/io/casehub/iot/mcp/IoTDeviceMcpToolTest.java` — adapt existing setUp, add 8 new tests

**Interfaces:**
- Consumes: `IoTRoles.VIEWER`, `IoTRoles.OPERATOR` (Task 1)
- Consumes: `DeviceRegistry.findById(String, String)` (Task 2)
- Consumes: `DeviceStateHistoryProvider.findHistory(String, String, Instant, Instant, int)` (Task 2)
- Consumes: `McpIdentityContext.tenancyId()`, `McpIdentityContext.actorId()` (Task 3)

- [ ] **Step 1: Write failing tests for tenancy filtering**

Add to `IoTDeviceMcpToolTest.java` using `ide_insert_member`. First, update `setUp()` to inject a mock `McpIdentityContext`:

```java
private McpIdentityContext identityContext;

@BeforeEach
void setUp() {
    registry = new MockDeviceRegistry();
    provider = new MockDeviceProvider();
    providers = mockInstance(provider);
    auditEvents = mockEvent();
    historyProviders = mockInstance(null);

    identityContext = mock(McpIdentityContext.class);
    when(identityContext.tenancyId()).thenReturn("test-tenant");
    when(identityContext.actorId()).thenReturn("test-user");

    tool = new IoTDeviceMcpTool(registry, providers, MAPPER,
                                 auditEvents, historyProviders, identityContext);
}
```

Then add the new tests:

```java
@Test
void getDevicesFiltersByTenancy() {
    registry.register(light());
    registry.register(thermostat());
    var result = tool.getDevices(null, null, null);
    // findByTenancyId returns only devices matching "test-tenant"
    assertThat(result).contains("light.living_room");
}

@Test
void getStateRejectsDeviceFromOtherTenant() {
    var device = light();
    registry.register(device);
    when(identityContext.tenancyId()).thenReturn("other-tenant");
    var result = tool.getState("light.living_room");
    assertThat(result).isEqualTo("Device not found: light.living_room");
}

@Test
void getStateAllowsSameTenantDevice() {
    registry.register(light());
    var result = tool.getState("light.living_room");
    assertThat(result).contains("light.living_room");
}

@Test
void sendCommandUsesActorId() {
    registry.register(light());
    providers = mockInstance(provider);
    tool = new IoTDeviceMcpTool(registry, providers, MAPPER,
                                 auditEvents, historyProviders, identityContext);
    tool.sendCommand("light.living_room", "turn_on", null);
    var dispatched = provider.lastDispatchedCommand();
    assertThat(dispatched.dispatchedBy()).isEqualTo("test-user");
}

@Test
void sendCommandRejectsDeviceFromOtherTenant() {
    registry.register(light());
    when(identityContext.tenancyId()).thenReturn("other-tenant");
    var result = tool.sendCommand("light.living_room", "turn_on", null);
    assertThat(result).isEqualTo("Failed: Device not found: light.living_room");
}

@Test
void getHistoryRejectsDeviceFromOtherTenant() {
    var historyProvider = mock(DeviceStateHistoryProvider.class);
    when(historyProvider.findHistory(eq("light.living_room"), eq("other-tenant"), any(), any(), anyInt()))
            .thenReturn(List.of());
    historyProviders = mockInstance(historyProvider);
    when(identityContext.tenancyId()).thenReturn("other-tenant");
    tool = new IoTDeviceMcpTool(registry, providers, MAPPER,
                                 auditEvents, historyProviders, identityContext);
    registry.register(light());
    var result = tool.getHistory("light.living_room", null, null, null);
    assertThat(result).contains("No history found");
}

@Test
void getHistoryAllowsSameTenantDevice() {
    var entry = new DeviceStateHistoryProvider.HistoryEntry(
            "light.living_room", "LIGHT", light(), List.of("state"), NOW);
    var historyProvider = mock(DeviceStateHistoryProvider.class);
    when(historyProvider.findHistory(eq("light.living_room"), eq("test-tenant"), any(), any(), anyInt()))
            .thenReturn(List.of(entry));
    historyProviders = mockInstance(historyProvider);
    tool = new IoTDeviceMcpTool(registry, providers, MAPPER,
                                 auditEvents, historyProviders, identityContext);
    registry.register(light());
    var result = tool.getHistory("light.living_room", null, null, null);
    assertThat(result).contains("light.living_room");
}

@Test
void getHistoryRejectsInvalidDateFormat() {
    var historyProvider = mock(DeviceStateHistoryProvider.class);
    historyProviders = mockInstance(historyProvider);
    tool = new IoTDeviceMcpTool(registry, providers, MAPPER,
                                 auditEvents, historyProviders, identityContext);
    var result = tool.getHistory("light.living_room", "not-a-date", null, null);
    assertThat(result).contains("Failed: Invalid date format");
}
```

- [ ] **Step 2: Run tests to verify new tests fail**

Run: `mvn --batch-mode test -pl mcp -Dtest=IoTDeviceMcpToolTest`
Expected: Compilation failure — constructor doesn't accept `McpIdentityContext` yet

- [ ] **Step 3: Add McpIdentityContext to IoTDeviceMcpTool constructor**

Use `ide_edit_member` to update the constructor in `IoTDeviceMcpTool.java`:

```java
@Inject
public IoTDeviceMcpTool(final DeviceRegistry deviceRegistry,
                        final Instance<DeviceProvider> providers,
                        final ObjectMapper objectMapper,
                        final Event<IoTCommandAuditEvent> auditEvents,
                        final Instance<DeviceStateHistoryProvider> historyProviders,
                        final McpIdentityContext identityContext) {
    this.deviceRegistry = deviceRegistry;
    this.providers = providers;
    this.objectMapper = objectMapper;
    this.auditEvents = auditEvents;
    this.historyProviders = historyProviders;
    this.identityContext = identityContext;
}
```

Add the field: `private final McpIdentityContext identityContext;`

- [ ] **Step 4: Add @RolesAllowed annotations**

Use `ide_edit_member` on each tool method to add annotations:

- `getDevices`: add `@RolesAllowed(IoTRoles.VIEWER)`
- `getState`: add `@RolesAllowed(IoTRoles.VIEWER)`
- `getHistory`: add `@RolesAllowed(IoTRoles.VIEWER)`
- `sendCommand`: add `@RolesAllowed(IoTRoles.OPERATOR)`

Add imports: `import jakarta.annotation.security.RolesAllowed;` and `import io.casehub.iot.api.IoTRoles;`

- [ ] **Step 5: Wire tenancy filtering in getDevices**

In `getDevices()`, replace:
```java
List<DeviceSummary> summaries = deviceRegistry.findAll().stream()
```
with:
```java
List<DeviceSummary> summaries = deviceRegistry.findByTenancyId(identityContext.tenancyId()).stream()
```

- [ ] **Step 6: Wire tenancy filtering in getState**

In `getState()`, replace:
```java
return deviceRegistry.findById(deviceId)
```
with:
```java
return deviceRegistry.findById(deviceId, identityContext.tenancyId())
```

- [ ] **Step 7: Wire tenancy filtering in sendCommand**

In `sendCommand()`, replace:
```java
var deviceOpt = deviceRegistry.findById(deviceId);
```
with:
```java
var deviceOpt = deviceRegistry.findById(deviceId, identityContext.tenancyId());
```

Replace `"mcp-agent"` with `identityContext.actorId()` in the DeviceCommand constructor.

- [ ] **Step 8: Wire tenancy in getHistory and add date validation**

Replace the `getHistory` method body to add tenancy pass-through and date validation:

```java
if (!historyProviders.isResolvable()) {
    return "Failed: Device state history is not available in this deployment.";
}

var historyProvider = historyProviders.get();
Instant fromInstant;
Instant toInstant;
try {
    fromInstant = from != null ? Instant.parse(from) : null;
    toInstant = to != null ? Instant.parse(to) : null;
} catch (DateTimeParseException e) {
    return "Failed: Invalid date format. Use ISO-8601 (e.g. '2026-07-01T00:00:00Z').";
}
int effectiveLimit = limit != null ? Math.min(limit, 200) : 50;

var entries = historyProvider.findHistory(deviceId, identityContext.tenancyId(),
                                          fromInstant, toInstant, effectiveLimit);

if (entries.isEmpty()) {
    return "No history found for device: " + deviceId;
}

try {
    return objectMapper.writeValueAsString(entries);
} catch (final JsonProcessingException e) {
    LOG.warnf("iot_get_history failed [%s]: %s",
            e.getClass().getSimpleName(), e.getMessage());
    return "Failed: " + e.getMessage();
}
```

Add import: `import java.time.format.DateTimeParseException;`

- [ ] **Step 9: Wire tenancyId in fireAuditEvent**

Replace the `"pending-identity-context"` placeholder in `fireAuditEvent`:

```java
private void fireAuditEvent(DeviceCommand command, CommandResult result, String providerId) {
    auditEvents.fireAsync(new IoTCommandAuditEvent(
            command.targetDeviceId(),
            command.action(),
            command.parameters(),
            result,
            command.dispatchedBy(),
            command.correlationId(),
            providerId,
            identityContext.tenancyId(),
            Instant.now()));
}
```

- [ ] **Step 10: Run all tests**

Run: `mvn --batch-mode test -pl mcp`
Expected: PASS — all existing tests (with updated setUp) + 8 new tests green

- [ ] **Step 11: Full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add mcp/
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(#74): RBAC, tenancy filtering, and principal propagation for MCP tools"
```
