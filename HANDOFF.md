# Handoff — casehub-iot

**Head commit (project):** 68b5031 — docs: sync ARC42STORIES.MD — stale scan at session wrap

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Run `/work` and begin #8 (YAML fixture loading for iot-testing). User queued three issues in sequence: #12 (done), #8, #11. Start #8 next.

## What's Left

- Minor code review items from C4 (#13) — HVAC Control+Switch mapping, missing OpenHabCommandDispatchTest, duplicate tagSet() utility, availability per-member vs per-device · S · Low
- Pre-existing HomeAssistantConfigTest failure — Quarkus startup failure in `@QuarkusTest`, blocks full `mvn install` across all modules · XS · Low
- PLATFORM.md casehub-iot deep-dive doc — `docs/repos/casehub-iot.md` noted as "pending — create after first module ships" · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #8 | YAML fixture loading for iot-testing | S | Low | Start here — next in queued sequence |
| #11 | Thing-scoped discovery fallback (OH) | M | Med | Third in queued sequence |
| #5 | Bridge Runtime — cloud/hybrid deployment | M | Med | Final chapter |

## Key References

- ARC42STORIES: `ARC42STORIES.MD` (project repo) — C1–C4 layers ✅, C5 🔲
- C4 design spec: `docs/superpowers/specs/2026-06-10-chapter4-openhab-provider-design.md`
- Basic auth spec: `docs/superpowers/specs/2026-06-12-openhab-basic-auth-design.md`
- Foundation spec: `docs/superpowers/specs/2026-06-05-iot-foundation-design.md`
- GitHub repo: `casehubio/iot`
