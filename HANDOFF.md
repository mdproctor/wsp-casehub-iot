# Handoff — casehub-iot

**Head commit (project):** 56d452a — feat(#69): add casehub-iot-mcp module — MCP tool surface for DeviceProvider operations

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

1. #63 (LLM resolution agent) — **unblocked** (from prior session)
2. #74 (RBAC, tenancy filtering, principal propagation for MCP tools) — **blocked on life#60 §8**
3. #46 (evaluate webapp extraction to app tier) — independent

## What Was Done (2026-07-26)

**Branch:** `issue-69-mcp-device-provider-tools` — closed, landed as `56d452a`.
**Covers:** #69.

- **casehub-iot-mcp module** — new library module exposing `iot_get_devices`,
  `iot_get_state`, `iot_send_command` as Quarkus MCP server tools. Follows the
  `connectors/mcp` library pattern. Host-agnostic: injects `DeviceRegistry` and
  `Instance<DeviceProvider>` — works in bridge (local HA/OpenHAB), webapp
  (remote via BridgeDeviceProvider), or both (federated).
- **DeviceSummary projection** — `iot_get_devices` returns summary fields only
  (deviceId, class, label, provider, available, lastUpdated) for token-efficient
  LLM context. `iot_get_state` returns the full Jackson-serialized `DeviceEntity`.
- **Native Map parameters** — `iot_send_command` takes `Map<String, Object>`
  via framework-native deserialization, avoiding double-encoded JSON strings.
- **30s dispatch timeout** — safety net for the MCP worker thread pool.
- **Design review:** 4 rounds, 11 issues, 11 verified. $14.69.
- **19 unit tests** covering all tools, filters, error paths, result variants.

## Cross-Repo Changes

- casehubio/parent#395 filed — sync platform docs (capability-ownership + repo doc) for casehub-iot-mcp.
- Engine branch `issue-62-cbr-summary-stats` (from prior session) — still not merged to engine main.
- RAS branch `issue-52-policy-decision-suppress` (from prior session) — still not merged to ras main.

## Deferred Issues

| # | Description | Blocked by |
|---|-------------|------------|
| #74 | RBAC, tenancy filtering, dispatchedBy principal propagation | life#60 §8 |
| #75 | Audit/ledger integration for MCP tool commands | — |
| #76 | Device state history queries via MCP | JPA dependency |
| #77 | WebSocket/SSE streaming of state changes via MCP | Future protocol |

## What's Left

- #68 — DismissalGangliaObserver: close ganglia on situation dismissal · XS · Low
- #66 — sync CBR situation resolution spec §4-5 (queue → subject view toolkit) · XS · Low

## Key References

- Spec: `specs/issue-69-mcp-device-provider-tools/2026-07-26-iot-mcp-tools-design.md`
- Design review: `~/adr/casehub-iot/iot-mcp-tools-*/tracker.md`
- Plan: `plans/attic/issue-69-mcp-device-provider-tools/2026-07-26-iot-mcp-tools.md`
- Garden: GE-20260726-523784 — `@ToolArg Map<String, Object>` native support
