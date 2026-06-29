# Handoff — casehub-iot

**Head commit (project):** 7fc5cfa — docs: add bridge-persistence modules to CLAUDE.md module table

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

No urgent work. Bridge persistence stack is feature-complete for current needs. Next discretionary work: native image Dockerfile or table partitioning (#42).

## What's Left

- casehubio/parent#325 — PLATFORM.md sync for BridgeAuditStore retention capability · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |

## Cross-Module

**We delivered** (other modules can now use):
- Configurable audit data retention via `retention-days` / `purge-interval` — disabled by default
- Type-safe JPA Criteria API via Hibernate metamodel (`BridgeAuditJpaEntity_`)
- PostgreSQL Testcontainers dialect validation test
- Protocol PP-20260629-31cb89: store-owned retention mechanism (garden)

## Key References

- Spec: `docs/superpowers/specs/2026-06-29-audit-retention-and-cleanup-design.md`
- GitHub repo: `casehubio/iot`
