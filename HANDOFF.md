# Handoff — casehub-iot

**Head commit (project):** 1f47ef5 — feat(homeassistant): Home Assistant provider — REST + WebSocket + supplement types #3

---

## What This Repo Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Immediate Next Step

Run `/work` and begin Chapter 4 (OpenHAB Provider, #4). Semantic model discovery from Equipment Groups, SSE subscription with 50ms coalescing window, item command dispatch. C3 HA provider is the reference implementation — same SPI, same patterns, higher complexity (state cache, semantic model).

## What's Left

- PLATFORM.md folder names + repo URL update in casehub-parent · S · Low — casehubio/parent#211
- MockReactiveDeviceProvider in iot-testing · S · Low — #9
- Minor code review items (DTO naming, FakeHaServer cleanup, test port, CloseReason) · XS · Low — #10
- Two unfiled forage entries: (1) Quarkus REST Client Reactive throws WebApplicationException on 4xx/5xx even with Uni<Response>; (2) Jackson @JsonProperty on records produces nested JSON — need @JsonAnyGetter for flat body · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #4 | OpenHAB Provider — REST + SSE, state cache, semantic model, supplement types | M | High | Start here — unblocks C5 |
| #5 | Bridge Runtime — cloud/hybrid deployment | M | Med | Depends on #3 or #4 |
| #8 | YAML fixture loading for iot-testing | S | Low | Deferred; Java fixtures sufficient |

## Key References

- ARC42STORIES: `ARC42STORIES.MD` (project repo) — C1 ✅, C2 ✅, C3 ✅ (issues closed, branch merged), C4–C5 🔲
- C3 design spec: `docs/superpowers/specs/2026-06-09-chapter3-homeassistant-provider-design.md`
- Foundation spec: `docs/superpowers/specs/2026-06-05-iot-foundation-design.md`
- GitHub repo: `casehubio/iot` (renamed from casehubio/casehub-iot this session)
