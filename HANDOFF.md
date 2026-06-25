# Handoff — casehub-iot

**Head commit (project):** 7e7db4e — docs: sync ARC42STORIES.MD — 10→11 typed subclasses after CameraDevice

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Repo is clean — zero open issues. Pick from What's Next or start casehub-life integration work.

## What's Left

None — all filed issues closed.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Server-side audit event log | M | Med | Persistent replayed events transit wire but aren't processed |
| — | Provider auto-discovery (mDNS/SSDP) | S | Med | Plug-and-play bridge deployment on Raspberry Pi |
| — | Bridge Docker Compose + deployment guide | S | Low | No "run this on your Pi" artifact yet |
| — | Qhorus CloudEvent adapter | S | Low | Filed: casehubio/qhorus#300 |
| — | Connectors CloudEvent adapter | S | Low | Filed: casehubio/connectors#23 |

## Cross-Module

**We delivered** (other modules can now use):
- CameraDevice type — casehub-life can observe camera streaming state
- Bridge health endpoint — operators can monitor bridge connectivity at `/q/health/ready`
- `casehub-ras` and any CloudEvent consumer can observe `@ObservesAsync CloudEvent` for IoT state changes

## Key References

- CameraDevice + bridge health + error handling spec: `docs/superpowers/specs/2026-06-25-camera-health-errfix-design.md`
- CloudEvent adapter spec: `docs/superpowers/specs/2026-06-20-cloudevent-adapter-design.md`
- Bridge spec: `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`
- ARC42STORIES: `ARC42STORIES.MD` — C1–C5 ✅, 11 device classes
- GitHub repo: `casehubio/iot`
