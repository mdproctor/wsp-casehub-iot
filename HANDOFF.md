# Handoff — casehub-iot

**Head commit (project):** 5a5917f — feat: CloudEvent adapter for StateChangeEvent — Closes #19

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Backlog clear. No open issues. Pick up new work or start an initiative — server-side audit event log is the next natural capability.

## What's Left

*None — backlog clear.*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Server-side audit event log | M | Med | Persistent replayed events transit wire but aren't processed — future audit pipeline |
| — | Qhorus CloudEvent adapter | S | Low | Follow same pattern as IoT adapter: `io.casehub.qhorus.message.<messageType>` — underscore convention established |
| — | Connectors CloudEvent adapter | S | Low | Same pattern: `io.casehub.connectors.inbound.<connectorType>` |
| casehubio/parent#290 | Sync PLATFORM.md for IoT CloudEvent adapter | XS | Low | Capability ownership + cross-repo dependency map |

## Cross-Module

**We delivered** (other modules can now use):
- `casehub-ras` and any CloudEvent consumer can observe `@ObservesAsync CloudEvent` for IoT state changes without coupling to iot-api types

## Key References

- CloudEvent adapter spec: `docs/superpowers/specs/2026-06-20-cloudevent-adapter-design.md`
- Bridge spec: `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`
- P0 layering decisions: `casehubio/parent — docs/superpowers/specs/2026-06-13-p0-layering-decisions-design.md`
- ARC42STORIES: `ARC42STORIES.MD` — C1–C5 ✅, journey complete
- GitHub repo: `casehubio/iot`
