# Handoff — casehub-iot

**Head commit (project):** a5533a6 — fix(api): add Jandex index for CDI bean discovery

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

All S/XS/M issues from the previous backlog are done. The branch is closed and merged. Pick new work from GitHub issues or start a new initiative.

## What's Left

- PLATFORM.md + deep-dive sync for bridge modules · XS · Low · casehubio/parent#264

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Server-side audit event log | M | Med | Persistent replayed events transit wire but aren't processed — future audit pipeline needs server-side storage |
| — | HomeAssistantProviderTest stabilisation | S | Med | MockServer integration tests have timeout flakiness — pre-existing |

## Key References

- Bridge spec: `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`
- Command dispatch spec: `docs/superpowers/specs/2026-06-17-bridge-command-dispatch.md`
- ThingType inference spec: `docs/superpowers/specs/2026-06-18-thing-type-device-class-inference.md`
- Store-and-forward spec: `docs/superpowers/specs/2026-06-18-durable-store-and-forward.md`
- ARC42STORIES: `ARC42STORIES.MD` — C1–C5 ✅, journey complete
- GitHub repo: `casehubio/iot`
