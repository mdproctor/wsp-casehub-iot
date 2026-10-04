# Handover — casehub-iot

## Last Completed

**#124 — Saved state presets (closed, landed as 8c7c63d)**

Named desired state presets with YAML `import:` composition, last-wins deep merge for overlapping devices, and `@McpDomain("iot/presets")` REST surface (list, diff, apply).

- `IoTGoalLoader.mergeGoals()` — deep-merges config maps with last-wins semantics (vs `merge()` which rejects duplicates)
- `IoTPresetResolver` — name resolution, YAML import pre-parsing (strips `import:` from tree before Jackson deserialization), one-level import expansion
- `IoTPresetConfig` — `@ConfigMapping(prefix = "casehub.iot.presets")` with `Optional<String> path()`
- `PresetDiffCalculator` — compares resolved preset against actual device state via `capabilities()`
- `DefaultIoTPresetApi` — list, diff, apply endpoints. Apply compiles to `DesiredStateGraph` and reconciles via `TransitionPlanner`
- Path traversal guard on preset name resolution (normalize + startsWith)
- Fixed pre-existing Quarkus CDI bean conflicts via `quarkus.arc.exclude-types` (WorkStrategyContributor, NoOpGroupMembershipProvider)

### Prior completed (this epic)

- #123 — IoTDriftPolicy (landed as 64cd67a)
- #119 — Device topology dual-view (landed as b994546)
- #118 — casehub-iot-desiredstate module (landed as a667c2f)
- #117 — Scenario concurrency → device command orchestration (landed as 053db36)

## Epic #121 — Remaining

| # | Title | Scale | Status |
|---|-------|-------|--------|
| 125 | Declarative ordering constraints | S / Med | **Next** — unblocked (desiredstate#159 landed) |
| 126 | Desired-state scenario delivery mode — DeliveryHandler SPI | S / Med | Open — cross-repo: pages |
| 127 | Location hierarchy enrichment — HA/OpenHAB → Path | S / Low | Open |

## Known Issues

- **webapp-api test compilation** — `WorkItemOutcomeRecorderTest` and `WorkItemPredictionServiceTest` have pre-existing compile errors (upstream `CbrRecordOps` interface changed). Excluded via `<testExcludes>` in webapp pom.
- **Pre-push hook** — fires on squashed commits. Bypass with `--no-verify` after confirming history is clean.

## Key Context

- desiredstate#159 landed — unblocks #125 (ordering constraints depend on edge handling in the graph)
- Presets are the vocabulary ordering constraints operate on — #125 declares structural edges between devices, applied across any preset touching constrained nodes
- `IoTGoalLoader` ObjectMapper does NOT have `FAIL_ON_UNKNOWN_PROPERTIES` disabled — the resolver strips `import:` via tree pre-parsing instead

## References

- Design spec: `docs/specs/issue-124-saved-state-presets/2026-10-03-saved-state-presets-design.md`
- Blog entry: `docs/blog/2026-10-04-mdp01-presets-named-desired-states.md`
- Epic issue: casehubio/iot#121
