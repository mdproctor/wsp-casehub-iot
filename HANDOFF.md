# Handover — casehub-iot

## Last Session

Implemented #125 — declarative ordering constraints for IoT desired state (landed as eca55ae). Full lifecycle: brainstormed 8 design decisions with standard adversarial review (3 HIGH findings caught and addressed), implemented 6 tasks across 4 batches, code review clean, branch audit clean, squashed and pushed.

Key design choice: composite NodeTypes (`device-config/light`, `physical-device/lock`) so the foundation `OrderingConstraint(NodeType, NodeType)` mechanism works at DeviceClass granularity without custom resolution logic. Two-constraint expansion (same-prefix only) — cross-prefix ordering handled by existing within-device dependency edges.

## Immediate Next Step

Pick up #127 (location hierarchy enrichment, S/Low) or #126 (delivery mode, S/Med — needs DeliveryHandler SPI in casehub-pages first).

## Known Issues

- **webapp-api test compilation** — `WorkItemOutcomeRecorderTest` and `WorkItemPredictionServiceTest` have pre-existing compile errors (upstream `CbrRecordOps` interface changed). Excluded via `<testExcludes>` in webapp pom.

## References

- Design spec: `docs/specs/issue-125-declarative-ordering-constraints/2026-10-04-declarative-ordering-constraints-design.md`
- Decisions: `docs/specs/issue-125-declarative-ordering-constraints/decisions.md`
- Epic issue: casehubio/iot#121 (2 remaining: #126, #127)
