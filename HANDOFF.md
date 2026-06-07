# Handoff — casehub-iot

**Head commit (project):** 27faf4a — docs: sync ARC42STORIES.MD — C1 complete

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Run `/work` and begin Chapter 2 (Test Infrastructure, #2) — `iot-testing` module. MockDeviceProvider, TestDeviceRegistry, StateChangeEventPublisher, fixture YAML. This unblocks casehub-life Layer 9 and enables TDD of providers in Chapters 3/4.

## What's Left

- Standardize tests on AssertJ + move assertj version to parent BOM · XS · Low — #7
- Rename submodule folders to drop `iot-` prefix · S · Low — #6

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | Test Infrastructure — MockDeviceProvider, fixtures, event publisher | S | Low | Start here — unblocks casehub-life Layer 9 |
| #3 | Home Assistant Provider — REST + WebSocket, supplement types | M | Med | Depends on #2 for TDD |
| #4 | OpenHAB Provider — REST + SSE, state cache, supplement types | M | High | Depends on #2; semantic model prerequisite |
| #5 | Bridge Runtime — cloud/hybrid deployment | M | Med | Depends on #3 or #4 |

## Key References

- ARC42STORIES: `ARC42STORIES.MD` (project repo) — C1 ✅, C2–C5 🔲
- C1 design spec: `docs/superpowers/specs/2026-06-07-chapter1-core-api-design.md`
- Foundation spec: `docs/superpowers/specs/2026-06-05-iot-foundation-design.md`
- Blog: `blog/2026-06-07-mdp01-chapter1-type-system.md` (workspace)
- Protocol updated: `garden/docs/protocols/universal/module-tier-structure.md` (CDI annotation pragmatism)
