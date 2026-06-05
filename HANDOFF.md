# Handoff — casehub-iot

**Head commit (project):** c8574fd — init: scaffold casehub-iot — CLAUDE.md, pom.xml, foundation design spec

---

## What This Repo Is

New foundation module — typed IoT device abstraction layer for Home Assistant, OpenHAB, and future IoT platforms. Peer to `casehub-connectors`. Consumed initially by `casehub-life` Layer 9.

## Immediate Next Step

Run `work-start` and begin implementing from the foundation design spec:  
`docs/superpowers/specs/2026-06-05-iot-foundation-design.md` in the project repo.

Start with `iot-api` — the public SPI module. Everything else depends on it.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Implement `iot-api` — DeviceEntity hierarchy, DeviceProvider SPI, StateChangeEvent, DeviceCommand, DeviceRegistry | M | Med | Start here — public API, semver from day one |
| — | Implement `iot-homeassistant` — HA REST + WebSocket provider, supplement types | M | Med | Depends on iot-api |
| — | Implement `iot-openhab` — OpenHAB REST + SSE provider, state cache, supplement types | M | High | Depends on iot-api; semantic model prerequisite |
| — | Implement `iot-testing` — MockDeviceProvider, fixture devices, TestStateChangePublisher | S | Low | Depends on iot-api |
| — | Implement `iot-bridge` — lightweight bridge runtime for cloud/hybrid mode | M | Med | Depends on iot-api + one provider |

## Key References

- Design spec: `docs/superpowers/specs/2026-06-05-iot-foundation-design.md` (project repo)
- Research: `casehubio/parent` — `docs/superpowers/research/2026-06-05-home-automation-research.md`
- Life Layer 9 spec (downstream): `casehubio/parent` — `docs/superpowers/specs/2026-06-05-life-layer9-home-automation.md`

## Notes

- `proj/` symlink needs to be created at session start: `ln -s /Users/mdproctor/claude/casehub/iot proj`
- Pre-push hook must be activated: `git config core.hooksPath .githooks` in the project repo
