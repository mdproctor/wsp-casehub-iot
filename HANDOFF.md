# Handoff — casehub-iot

**Head commit (project):** 5ab6df6 — feat: Docker Compose + deployment guide — #32

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
| #35 | BridgeAuditStore SPI for structured audit query | M | Med | Deferred from #34 — build when a consuming app has concrete query need |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |
| — | ARC42STORIES.MD update for deployment + discovery + audit | S | Low | §6 Runtime, §7 Deployment, §8 Crosscutting need new sections |

## Cross-Module

**We delivered** (other modules can now use):
- Docker image `ghcr.io/casehubio/iot-bridge` — ARM64 + x86_64, deploy on Pi with `docker compose up -d`
- mDNS/SSDP auto-discovery — providers find HA/OpenHAB without explicit URL config
- `BridgeAuditEvent` CDI events — consuming apps observe `@ObservesAsync BridgeAuditEvent` for audit trail
- `@LookupIfProperty` provider activation — both providers in one Docker image, each enabled independently
- Single `casehub.iot.tenancy-id` — no more per-module tenancyId divergence

## Key References

- Design spec: `docs/superpowers/specs/2026-06-25-bridge-ops-and-audit-design.md`
- Deployment guide: `bridge/DEPLOYMENT.md`
- Diary: `blog/2026-06-26-mdp06-three-walls-between-config-and-runtime.md`
- GitHub repo: `casehubio/iot`
