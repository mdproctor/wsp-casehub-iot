# Handoff — casehub-iot

**Head commit (project):** 1a14a12 — feat: populate case file with device metadata for CBR feature extraction (#57)

---

## What This Repo Is

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Pick up CBR retrieval work. #50 (situation resolution suggestion via CBR) is the natural next — it consumes the feature extraction pipeline built in #49 and #57.

## What Was Done (2026-07-13)

**Branch:** `issue-57-case-file-device-metadata` — closed, landed as `1a14a12`.

- **CaseInputContributor SPI** — new extension point in `casehub-ras-api` (casehubio/casehub-ras#37, `e2d3770` on ras main). `DefaultCaseTrigger` discovers implementations via CDI and merges their contributions into the case input map.
- **IoTCaseInputContributor** — resolves deviceId from correlationKey (`device/<id>`), looks up DeviceEntity from DeviceRegistry, contributes `deviceClass`, `roomType`, `eventTimestamp` to the case working layer.
- **DeviceEntity.location** — nullable String field added. OpenHAB wires from `thing.location()`. HA area registry integration pending.
- **DeviceResponse** — now exposes `location` (was hardcoded `null`).

## Cross-Repo Changes

- **casehub-ras** — `e2d3770` on main: `CaseInputContributor` SPI + `DefaultCaseTrigger` integration. casehubio/casehub-ras#37 closed.

## What's Left

- #58 — CBR retention/purge policy · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | Situation resolution suggestion via CBR | M | Med | Consumes #49 + #57 |
| #51 | Work item outcome prediction via CBR | M | Med | Consumes #49 + #57 |
| #52 | False-positive suppression via CBR | M | Med | Consumes #49 + #57 |
| #46 | Evaluate webapp extraction to app tier | M | Med | |

## Key References

- Spec: `docs/superpowers/specs/2026-07-12-cbr-infrastructure-design.md`
- Garden: GE-20260713-b879b2 (H2 JSONB gotcha)
