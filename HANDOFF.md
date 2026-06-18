# Handoff — casehub-iot

**Head commit (project):** 0c84354 — fix(ci): add GitHub Packages config and replace MockWebServer with JDK HttpServer #24

---

## What This Repo Is

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## Immediate Next Step

CI is green. One open issue: **#19 — CloudEvent adapter for StateChangeEvent**. Pick it up with `/work`, or start a new initiative.

## What's Left

*None — backlog clear.*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #19 | CloudEvent adapter for StateChangeEvent | S | Med | Open issue on casehubio/iot |
| — | Server-side audit event log | M | Med | Persistent replayed events transit wire but aren't processed — future audit pipeline |
| — | HomeAssistantProviderTest stabilisation | — | — | **DONE** — replaced MockWebServer with JDK HttpServer; all 68 tests green in 3s |

## Key References

- Bridge spec: `docs/superpowers/specs/2026-06-16-bridge-runtime-design.md`
- Command dispatch spec: `docs/superpowers/specs/2026-06-17-bridge-command-dispatch.md`
- ThingType inference spec: `docs/superpowers/specs/2026-06-18-thing-type-device-class-inference.md`
- Store-and-forward spec: `docs/superpowers/specs/2026-06-18-durable-store-and-forward.md`
- ARC42STORIES: `ARC42STORIES.MD` — C1–C5 ✅, journey complete
- GitHub repo: `casehubio/iot`
