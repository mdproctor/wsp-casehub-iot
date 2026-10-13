## D1: Epic scope — operational console

**Choice:** Operational console — run, monitor, browse, explore. No visual scenario authoring (PagesBuilderShell) or CRUD (ScriptRegistry REST) in this epic.
**Alternatives:**
- Authoring + operational — includes visual editor and CRUD. Deferred: authoring is a separate concern with its own UX complexity.
- Minimal wiring only — just catalog + library panels. Too incremental to deliver an integrated experience.
**Rationale:** The operational console delivers end-to-end value (browse → select → run → monitor → inspect) without the authoring complexity. Authoring follows naturally once the operational surface proves the composition.
**Trade-offs:** Users can't create new playbooks from the UI — only run bundled ones. Acceptable for the current pre-release stage.
**Sources:** fsitrading ops-centre.ts (reference composition pattern), #128 playbook UI evaluation spec
**Exploration:** quick
**Status:** captured

## D2: Presets are a separate epic

**Choice:** Keep #136 (preset composer) and #137 (preset diff panel) out of this epic.
**Alternatives:**
- Include presets — presets are part of the operational experience. Deferred: preset UX is a distinct domain (device picker, state configuration, diff rendering) that doesn't share components with the playbook console.
**Rationale:** Presets use different platform components and have their own UX flow. Mixing them bloats the epic without adding to the playbook execution story.
**Trade-offs:** Operators can't compose presets from the console. They can still apply presets via playbooks that use `delivery: desired-state`.
**Sources:** #136, #137, DesiredStateDeliveryHandler.java
**Exploration:** quick
**Status:** captured

## D3: Keep playbook controller variants

**Choice:** Keep the existing `PagesPlaybookController` / `PagesPlaybookYamlViewer` (playbook variants). Rename the page from "Scenarios" to "Playbooks" for consistency.
**Alternatives:**
- Switch to scenario variants (PagesScenarioController) — semantically correct naming but requires rewiring REST endpoints and verifying feature parity. Risk of regressions with no functional gain.
**Rationale:** The playbook variants are already wired and working. The `/playbook/*` REST endpoints are established. Renaming the page to "Playbooks" aligns the user-facing label with the internal terminology.
**Trade-offs:** None — the "scenario" variants are a parallel implementation with the same features under different endpoint paths.
**Sources:** scenarios.ts, PagesPlaybookController, IoTPlaybookConfigProducer.java
**Exploration:** quick
**Status:** captured
