# RBAC, Tenancy Filtering, and Principal Propagation for MCP Tools

**Issue:** casehubio/iot#74
**Date:** 2026-08-02
**Status:** Draft
**Predecessor:** casehubio/iot#69 (MCP tool exposure — §8 scoped this work out)

---

## 1. Purpose

Add role-based access control, tenant-scoped data filtering, and principal
identity propagation to the four MCP tools in `casehub-iot-mcp`. Currently
all tools are unauthenticated, return all devices regardless of tenant, and
hardcode `"mcp-agent"` as the command dispatcher identity.

After this change:
- Read tools require `iot-viewer` role; command tools require `iot-operator`
- Device queries return only devices belonging to the caller's tenant
- Command audit trails record the actual caller identity

---

## 2. Constraints

- The MCP module is a **library** consumed by two hosts with different
  security postures:
  - **Webapp** — OIDC, multi-tenant, has `CurrentPrincipal` via casehub-work
  - **Bridge** — local, single-tenant, no OIDC, no `CurrentPrincipal`
- The design must not break the bridge. Zero changes required in bridge.
- `DeviceRegistry.findByTenancyId(String)` already exists — use it.
- `CurrentPrincipal` is a platform SPI (`casehub-platform-api`) — do not
  create IoT-specific principal abstractions.
- Single tenancy property: `casehub.iot.tenancy-id` (CLAUDE.md rule).

---

## 3. IoTRoles Constants

New class in `casehub-iot-api`:

```java
package io.casehub.iot.api;

public final class IoTRoles {
    public static final String VIEWER = "iot-viewer";
    public static final String OPERATOR = "iot-operator";
    public static final String ADMIN = "iot-admin";
    private IoTRoles() {}
}
```

Placed in the API module because host apps need these constants to
configure their OIDC role mappings.

**ARC42STORIES update:** §2 Constraints currently states
"Authorization-agnostic: `iot-api` does not enforce authorization."
This constraint remains true — `IoTRoles` defines vocabulary, not
enforcement. The constraint will be refined to: "Authorization-agnostic:
`iot-api` does not enforce authorization; it may define role name
constants used by consuming modules for annotation-based access control."

**Webapp migration:** `DeviceResource` currently hardcodes
`@RolesAllowed("iot-viewer")` and `@RolesAllowed("iot-operator")` as
string literals. `SituationResource` hardcodes `@RolesAllowed("iot-admin")`
on three mutation endpoints. All will be updated to use `IoTRoles`
constants (`IoTRoles.VIEWER`, `IoTRoles.OPERATOR`, `IoTRoles.ADMIN`)
to consolidate role name definitions.

---

## 4. Principal Resolution

### 4.1 McpIdentityContext

New `@ApplicationScoped` bean in `casehub-iot-mcp` encapsulating
principal resolution for the dual-host environment. The tool class
should not know about CDI container lifecycle states — that is
infrastructure plumbing, not tool behavior.

```java
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

`Instance<CurrentPrincipal>` (not direct `@Inject`) — defers resolution
so the bean is optional. Direct injection would cause
`UnsatisfiedResolutionException` at startup in hosts without a
`CurrentPrincipal` implementation (bridge).

Three-guard pattern in `isPrincipalAvailable()` (validated by
GE-20260627-f3476f):
1. `isResolvable()` — bean exists and is unambiguous
2. `Arc.container() != null` — CDI container running (fails in unit tests)
3. `requestContext().isActive()` — request scope live on this thread

`isPrincipalAvailable()` is package-private — test subclasses override
it to bypass the `Arc.container()` guard (see §6.2).

**Fallback behavior (bridge / background contexts):**
- Tenancy → `casehub.iot.tenancy-id` config property
- Actor → `"mcp-agent"` literal

### 4.2 Tool Class Injection

The tool class injects `McpIdentityContext` as a single dependency:

```java
private final McpIdentityContext identityContext;
```

No CDI introspection in the tool class — principal resolution is fully
encapsulated in `McpIdentityContext`.

---

## 5. Tool Method Changes

### 5.1 Role Annotations

| Tool | Annotation |
|------|-----------|
| `iot_get_devices` | `@RolesAllowed(IoTRoles.VIEWER)` |
| `iot_get_state` | `@RolesAllowed(IoTRoles.VIEWER)` |
| `iot_get_history` | `@RolesAllowed(IoTRoles.VIEWER)` |
| `iot_send_command` | `@RolesAllowed(IoTRoles.OPERATOR)` |

`@RolesAllowed` is a CDI interceptor — enforcement requires the host app
to have a security extension (`quarkus-oidc`, `quarkus-security`). In hosts
without one (bridge), the annotations are inert.

### 5.2 Tenancy Filtering

**SPI addition:** Add `Optional<DeviceEntity> findById(String deviceId,
String tenancyId)` to `DeviceRegistry`. Returns the device only if it
exists and its `tenancyId` matches the given tenant. Returns
`Optional.empty()` for cross-tenant devices — preserves the "don't leak
existence" invariant. The existing `findById(String)` remains for
non-tenancy contexts (bridge).

Note: `DeviceEntity`'s constructor enforces
`Objects.requireNonNull(tenancyId)` — tenancyId is never null.
No null-tenancy code path is needed. The existing
`DeviceResource.filterByTenancy()` null check is also dead and will
be simplified in the same change.

**`iot_get_devices`:** Replace `deviceRegistry.findAll()` with
`deviceRegistry.findByTenancyId(identityContext.tenancyId())`. Existing
filters (deviceClass, providerId, available) chain after.

**`iot_get_state`, `iot_send_command`:** Replace
`deviceRegistry.findById(deviceId)` with
`deviceRegistry.findById(deviceId, identityContext.tenancyId())`.
Cross-tenant devices return `Optional.empty()` → `"Device not found"`.
No inline tenancy check needed — the registry enforces it.

**`iot_get_history`:** The current implementation calls
`historyProviders.get().findHistory(deviceId, ...)` directly — there is
no `findById()` call to attach a tenancy check to. Add a device lookup
before the history query:

```java
var deviceOpt = deviceRegistry.findById(deviceId, identityContext.tenancyId());
if (deviceOpt.isEmpty()) {
    return "Device not found: " + deviceId;
}
```

This check goes before the `historyProviders.isResolvable()` guard —
tenancy rejection should not reveal whether a history provider exists.

The `DeviceStateHistoryProvider` SPI remains tenancy-unaware. The
device-level guard is sufficient: history entries are scoped by
deviceId, and the device lookup ensures the deviceId belongs to the
caller's tenant. The webapp's `DeviceResource.history()` has an
additional JPA-level tenancy filter (`AND h.tenancyId = :tenancyId`)
as defense-in-depth — this asymmetry is acceptable because devices
do not change tenants at runtime.

**Cross-tenant admin:** `CurrentPrincipal.isCrossTenantAdmin()` is not
consulted. A cross-tenant admin using MCP tools sees only their own
tenant's devices — consistent with the webapp's `DeviceResource` which
also uses strict tenancy equality (`filterByTenancy()`). Cross-tenant
MCP access is deferred to a future issue.

### 5.3 Actor Identity

In `iot_send_command`, replace:
```java
"mcp-agent"
```
with:
```java
identityContext.actorId()
```

The audit event then records the authenticated caller when available.

---

## 6. Testing

Unit tests only — no `@QuarkusTest`. `@RolesAllowed` enforcement is a
Quarkus framework concern tested in host apps.

### 6.1 New Tests

**McpIdentityContextTest:**

| Test | Verifies |
|------|----------|
| `tenancyIdReturnsPrincipalTenancy` | With principal → returns `principal.tenancyId()` |
| `tenancyIdFallsBackToConfig` | Without principal → returns `configTenancyId` |
| `actorIdReturnsPrincipalActor` | With principal → returns `principal.actorId()` |
| `actorIdFallsBackToMcpAgent` | Without principal → returns `"mcp-agent"` |

**IoTDeviceMcpToolTest:**

| Test | Verifies |
|------|----------|
| `getDevicesFiltersByTenancy` | Uses `findByTenancyId(identityContext.tenancyId())` |
| `getStateRejectsDeviceFromOtherTenant` | Wrong tenant → `"Device not found"` |
| `getStateAllowsSameTenantDevice` | Same tenant → returns state |
| `sendCommandUsesActorId` | `dispatchedBy` = `identityContext.actorId()` |
| `sendCommandRejectsDeviceFromOtherTenant` | Cross-tenant command → `"Device not found"` |
| `getHistoryRejectsDeviceFromOtherTenant` | Cross-tenant history → `"Device not found"` |
| `getHistoryAllowsSameTenantDevice` | Same tenant → returns history entries |

### 6.2 Test Setup

**McpIdentityContext tests:** Constructor-inject a mock
`Instance<CurrentPrincipal>` and `configTenancyId`:
- **With principal:** Subclass `McpIdentityContext` and override
  `isPrincipalAvailable()` to return `true`. Mock
  `Instance<CurrentPrincipal>` with `isResolvable()` → true, `get()`
  → stub with fixed tenancyId and actorId.
- **Without principal:** Construct directly — `Arc.container()` returns
  null in unit tests, so `isPrincipalAvailable()` naturally returns
  false and the fallback path is taken.

`isPrincipalAvailable()` is package-private, enabling test subclassing
without exposing CDI lifecycle internals to the public API.

**Tool class tests:** Mock `McpIdentityContext` — two methods
(`tenancyId()`, `actorId()`), clear contract. No CDI container
introspection or `Instance<>` mock construction needed in tool tests.

---

## 7. Dependencies

**`casehub-iot-api`:** No new dependencies. `IoTRoles` is a pure
constants class.

**`casehub-iot-mcp` (pom.xml):** `jakarta.annotation-api` (via
`quarkus-mcp-server-core`) and `io.quarkus:quarkus-arc` (via
`casehub-platform-api`) are transitively available. Both transitive paths
are stable — no explicit declarations needed.

---

## 8. Host App Impact

| Host | Security stack | Effect |
|------|---------------|--------|
| Webapp | OIDC + casehub-work (`TenantScopedPrincipal`) | Full RBAC + tenancy. Must map OIDC groups → `iot-viewer` / `iot-operator`. |
| Bridge | None | `@RolesAllowed` inert. `Instance<CurrentPrincipal>` unresolvable. Falls back to config tenancy + `"mcp-agent"`. **Zero changes.** |

---

## 9. Files Changed

| File | Change |
|------|--------|
| `api/.../IoTRoles.java` | New — role constants |
| `api/.../spi/DeviceRegistry.java` | Add `findById(String, String)` — tenancy-scoped lookup |
| `api/.../spi/CdiDeviceRegistry.java` | Implement `findById(String, String)` |
| `mcp/.../McpIdentityContext.java` | New — principal resolution bean |
| `mcp/.../McpIdentityContextTest.java` | New — tests for three-guard resolution |
| `mcp/.../IoTDeviceMcpTool.java` | Annotations, `McpIdentityContext` injection, tenancy filtering, actorId |
| `mcp/.../IoTDeviceMcpToolTest.java` | 6 new tenancy/identity tests |
| `webapp/.../DeviceResource.java` | Migrate `@RolesAllowed` string literals to `IoTRoles` constants |
| `webapp/.../SituationResource.java` | Migrate `@RolesAllowed("iot-admin")` to `IoTRoles.ADMIN` |
| `testing/.../MockDeviceRegistry.java` | Implement `findById(String, String)` |
| `ARC42STORIES.MD` | Refine authorization-agnostic constraint |
| `mcp/pom.xml` | No changes needed — transitive deps are stable |

---

## 10. Garden Context

Entries consulted during design:

| Entry | Relevance |
|-------|-----------|
| GE-20260627-f3476f | `Arc.container().requestContext().isActive()` pattern — adopted for principal resolution |
| GE-20260628-919f9f | Non-OIDC SecurityIdentity + CurrentPrincipal — validates the `Instance<>` approach for mixed-auth hosts |
| GE-20260623-941ade | `policy=deny` on `/*` blocks ALL requests — avoid HTTP-layer deny, use `@RolesAllowed` |
| GE-20260623-5b192f | `DenyUnannotatedPredicate` scope limitation — host apps should annotate all MCP tool classes |
| GE-20260719-4e2784 | `@TestSecurity` doesn't populate `CurrentPrincipal.groups()` — relevant for future host-app integration tests |
| GE-20260626-c94109 | `@LookupIfProperty` pattern — already used in provider activation, not needed here |

---

## 11. Not In Scope

- OIDC role mapping configuration in webapp — host app concern
- MCP transport authentication mechanism — host app concern
- Audit / ledger integration for tool commands — iot#75
- Device state history queries via MCP — already shipped in #76
- Cross-tenant admin access via MCP tools — deferred (webapp also
  ignores `isCrossTenantAdmin()` in `DeviceResource.filterByTenancy()`)
- WebSocket/SSE streaming — iot#77