# Handoff — casehub-iot

*Updated: parent#290 closed — removed from backlog.*

**Head commit (project):** 7e127b5 — docs: sync ARC42STORIES.MD — CloudEvent adapter, bridge-server, quarkus-jackson — Closes #27

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Close #24 and #25 on GitHub — commits landed on main but issues weren't auto-closed (used `#N` not `Closes #N`).

## What's Left

- Close #24 and #25 on GitHub — housekeeping · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Server-side audit event log | M | Med | Persistent replayed events transit wire but aren't processed |
| — | Qhorus CloudEvent adapter | S | Low | Same pattern as IoT: `io.casehub.qhorus.message.<messageType>` |
| — | Connectors CloudEvent adapter | S | Low | Same pattern: `io.casehub.connectors.inbound.<connectorType>` |

## Cross-Module

**We delivered** (other modules can now use):
- `casehub-ras` and any CloudEvent consumer can observe `@ObservesAsync CloudEvent` for IoT state changes without coupling to iot-api types

## Key References

- CloudEvent adapter spec: `docs/superpowers/specs/2026-06-20-cloudevent-adapter-design.md`
- Bridge spec: `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`
- ARC42STORIES: `ARC42STORIES.MD` — C1–C5 ✅, journey complete
- GitHub repo: `casehubio/iot`
