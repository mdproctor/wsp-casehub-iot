# Handoff — casehub-iot

*Updated: #37, casehubio/parent#317 closed — removed from backlog.*

**Head commit (project):** e0e7131 — feat: JPA BridgeAuditStore — durable audit persistence with JSONB message storage — Closes #38

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

No urgent work. #37 (reactive variant) was analysed and deferred — no reactive consumer exists. Next discretionary work: native image Dockerfile or data retention strategy (#40).

## What's Left

None — all filed issues are deferred by design.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #40 | BridgeAuditStore data retention strategy | S | Med | TTL cleanup, scheduled purge, or partitioning — deployment-specific |
| #41 | bridge-persistence minor review findings | XS | Low | Test assertion, quarkus-junit consistency, metamodel, Testcontainers |
| — | Native image Dockerfile (GraalVM) | M | High | Reflection config for DeviceTypeIdResolver needed |

## Cross-Module

**We delivered** (other modules can now use):
- `casehub-iot-bridge-persistence-jpa` — add to classpath for durable JPA audit with JSONB message storage
- `casehub-iot-bridge-persistence-memory` — in-memory ring buffer, `@Alternative @Priority(100)`, for Pi and test isolation
- `BridgeAuditQuery.offset` — pagination ready for any store implementation

## Key References

- Spec: `docs/superpowers/specs/2026-06-29-jpa-bridge-audit-store-design.md`
- Plan: `docs/superpowers/plans/2026-06-29-jpa-bridge-audit-store.md`
- GitHub repo: `casehubio/iot`
