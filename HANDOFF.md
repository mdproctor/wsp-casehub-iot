# Handoff — casehub-iot

**Head commit (project):** 8707354 — feat(#78,#79,#75,#76,#68,#66): SPI blocking migration, MCP audit/history, DismissalGangliaObserver, spec sync

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

1. #63 (LLM resolution agent — IoTAiResolutionAgent) — **unblocked**, top priority
2. File ras issue for full Ganglion Uni→blocking migration (detect/compact/close)
3. #46 (evaluate webapp extraction to app tier) — independent

## What Was Done (2026-07-27)

**Branch:** `issue-78-spi-blocking-and-batch` — closed, landed as `8707354`.
**Covers:** #78, #79, #75, #76, #68, #66.

- **#78+#79 SPI blocking migration** — DeviceProvider (discover, dispatch) and
  DeviceRegistry (refresh) migrated from Uni to blocking. All implementations,
  REST clients, callers, and tests updated. BridgeDeviceProvider uses
  CompletableFuture instead of UniEmitter.
- **#75 MCP audit events** — IoTCommandAuditEvent CDI event fired on every
  iot_send_command dispatch (success and failure).
- **#76 MCP device history** — iot_get_history tool with DeviceStateHistoryProvider
  SPI. JPA implementation in webapp. Optional inject for graceful fallback.
- **#68 DismissalGangliaObserver** — observes DISMISSED events, closes referenced
  ganglia. findBySituationId() added to SituationDefinitionRegistry (ras).
- **#66 Spec sync** — CBR spec §4-5 rewritten: platform-queue → subject view toolkit.

## Cross-Repo Changes

- **ras:** RasTriggerPolicy migrated to blocking. findBySituationId() added.
  Cherry-picked to ras main (5007cc3). Pushed to origin.
- **ops:** IoTNodeProvisioner caller updated (d8b0615 on ops main). Pushed.
- casehubio/parent#395 (from prior session) — still open.

## What's Left

- File ras issue for full Ganglion Uni→blocking migration · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #63 | LLM resolution agent — IoTAiResolutionAgent | L | High | Unblocked, top priority |
| #74 | RBAC, tenancy filtering, principal propagation for MCP | M | Med | Blocked on life#60 §8 |
| #77 | WebSocket/SSE streaming state changes via MCP | M | High | Future protocol |
| #46 | Evaluate webapp extraction to app tier | L | High | Independent |

## Key References

- Blog: `blog/2026-07-27-mdp01-six-issues-one-branch.md`
- Garden: GE-20260727-300281 — IntelliJ MCP truncation on timeout
