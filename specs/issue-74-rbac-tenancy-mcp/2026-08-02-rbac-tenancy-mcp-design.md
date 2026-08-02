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
    private IoTRoles() {}
}
```

Placed in the API module because host apps need these constants to
configure their OIDC role mappings.

---

## 4. Principal Resolution

### 4.1 New Injections

```java
private final Instance<CurrentPrincipal> currentPrincipal;

@ConfigProperty(name = "casehub.iot.tenancy-id")
String configTenancyId;
```

`Instance<CurrentPrincipal>` (not direct `@Inject`) — defers resolution
so the bean is optional. Direct injection would cause
`UnsatisfiedResolutionException` at startup in hosts without a
`CurrentPrincipal` implementation (bridge).

### 4.2 Helper Methods

```java
private String resolveTenancyId() {
    if (currentPrincipal.isResolvable()
            && Arc.container() != null
            && Arc.container().requestContext().isActive()) {
        return currentPrincipal.get().tenancyId();
    }
    return configTenancyId;
}

private String resolveActorId() {
    if (currentPrincipal.isResolvable()
            && Arc.container() != null
            && Arc.container().requestContext().isActive()) {
        return currentPrincipal.get().actorId();
    }
    return "mcp-agent";
}
```

Three-guard pattern (validated by GE-20260627-f3476f):
1. `isResolvable()` — bean exists and is unambiguous
2. `Arc.container() != null` — CDI container running (fails in unit tests)
3. `requestContext().isActive()` — request scope live on this thread

**Fallback behavior (bridge / background contexts):**
- Tenancy → `casehub.iot.tenancy-id` config property
- Actor → `"mcp-agent"` literal

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

**`iot_get_devices`:** Replace `deviceRegistry.findAll()` with
`deviceRegistry.findByTenancyId(resolveTenancyId())`. Existing filters
(deviceClass, providerId, available) chain after.

**`iot_get_state`, `iot_get_history`, `iot_send_command`:** After
`deviceRegistry.findById(deviceId)`, verify device tenancy:

```java
if (!device.tenancyId().equals(resolveTenancyId())) {
    return "Device not found: " + deviceId;
}
```

Returns "Device not found" on tenancy mismatch — does not leak that the
device exists in another tenant. Same error message as a genuinely
missing device.

### 5.3 Actor Identity

In `iot_send_command`, replace:
```java
"mcp-agent"
```
with:
```java
resolveActorId()
```

The audit event then records the authenticated caller when available.

---

## 6. Testing

Unit tests only — no `@QuarkusTest`. `@RolesAllowed` enforcement is a
Quarkus framework concern tested in host apps.

### 6.1 New Tests

| Test | Verifies |
|------|----------|
| `getDevicesFiltersByTenancy` | With principal → uses `findByTenancyId(principal.tenancyId())` |
| `getDevicesFallsBackToConfigTenancy` | Without principal → uses `findByTenancyId(configTenancyId)` |
| `getStateRejectsDeviceFromOtherTenant` | Wrong tenant → `"Device not found"` |
| `getStateAllowsSameTenantDevice` | Same tenant → returns state |
| `sendCommandUsesActorIdFromPrincipal` | With principal → `dispatchedBy` = `actorId()` |
| `sendCommandFallsBackToMcpAgent` | Without principal → `dispatchedBy` = `"mcp-agent"` |
| `sendCommandRejectsDeviceFromOtherTenant` | Cross-tenant command → `"Device not found"` |
| `getHistoryRejectsDeviceFromOtherTenant` | Cross-tenant history → `"Device not found"` |

### 6.2 Test Setup

Constructor-inject a mock `Instance<CurrentPrincipal>`:
- **With principal:** `isResolvable()` returns true, `get()` returns a stub
  with fixed tenancyId and actorId
- **Without principal:** `isResolvable()` returns false

`configTenancyId` set via reflection or package-private access. No CDI
container needed — `Instance<>` is a mockable interface.

The `Arc.container()` guard returns null in unit tests, so the
`isResolvable()` check is sufficient — the scope check is never reached.

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
| `mcp/.../IoTDeviceMcpTool.java` | Annotations, principal injection, tenancy filtering, actorId |
| `mcp/.../IoTDeviceMcpToolTest.java` | 8 new tests |
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
- Device state history queries via MCP — already shipped in #69
- WebSocket/SSE streaming — iot#77
