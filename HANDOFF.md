# Handoff — casehub-iot

**Head commit (project):** 22ac5ee — docs: sync ARC42STORIES.MD — C2 complete, reversals documented

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Run `/work` and begin Chapter 3 (Home Assistant Provider, #3). REST discovery via `/api/states`, WebSocket subscription with changedCapabilities diffing, service call command dispatch, connection lifecycle with exponential backoff. `iot-testing` (C2) is now available for TDD.

## What's Left

- Rename submodule folders to drop `iot-` prefix · S · Low — #6
- Standardize remaining test assertions on AssertJ (assertj-core BOM move done; ExtensibleDeviceTest still uses JUnit assertions) · XS · Low — #7

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | Home Assistant Provider — REST + WebSocket, supplement types | M | Med | Start here — unblocks C4/C5 |
| #4 | OpenHAB Provider — REST + SSE, state cache, supplement types | M | High | Depends on #2; semantic model prerequisite |
| #5 | Bridge Runtime — cloud/hybrid deployment | M | Med | Depends on #3 or #4 |
| #8 | YAML fixture loading for iot-testing — additive, Java factories are primary | S | Low | Deferred; Java fixtures are sufficient |

## Key References

- ARC42STORIES: `ARC42STORIES.MD` (project repo) — C1 ✅, C2 ✅, C3–C5 🔲
- C2 design spec: `docs/superpowers/specs/2026-06-08-chapter2-test-infrastructure-design.md`
- C2 blog: `blog/2026-06-09-mdp02-chapter2-test-infrastructure.md` (workspace)
- Foundation spec: `docs/superpowers/specs/2026-06-05-iot-foundation-design.md`
- Garden: GE-20260609-8f14bb — Java records + BigDecimal scale-sensitive equals()
