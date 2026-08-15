# Handoff — casehub-iot

**Head commit (project):** 2d18799 — docs: sync ARC42STORIES.MD — stale scan at session wrap
**Date:** 2026-08-15

---

## Last Session

Audited the webapp's frontend integration against casehub-pages and blocks-ui. Found blocks-ui is declared as a Maven dependency but never imported — all 7 pages build UI manually with pages-ui DSL. Mapped 40+ blocks-ui components against the pages, identified 3 high-impact and 2 medium-impact replacements. Filed #95 to track the work. Arc42 stale scan fixed 3 items (#35, #20/#22, #11).

## Immediate Next Step

Pick up #82 (CBR re-routing on context changes) — next on the critical path. Run `/work` to begin. #95 (blocks-ui adoption) is available but lower priority.

## Cross-Repo

**Blocking** (we owe):
- engine#834 — Ganglion Uni→blocking migration · M · Med

**Filed** (engine owes us):
- engine#872 — Token usage exposure from Agent.execute() · S · Low — needed for #85 token metrics
