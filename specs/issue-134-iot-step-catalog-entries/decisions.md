## D1: YAML catalog file following fsitrading pattern

**Choice:** Single `iot-steps.yaml` file in `webapp/src/main/resources/steps/` with namespace `iot`, defining three actions (`command`, `state`, `desired-state`). Action names match existing `@Plugin` names when namespace-qualified. MCP invoke bindings for the two plugin steps; metadata-only for the delivery handler.
**Alternatives:**
- Java-only (no YAML) — rely solely on APT-generated `META-INF/yaml-plugins/` manifests. Misses `desired-state` entirely and provides less rich metadata for the UI catalog.
- Separate YAML files per step — unnecessary fragmentation for 3 entries.
**Rationale:** The fsitrading reference implementation establishes the pattern. YAML gives richer UI catalog metadata (descriptions, defaults, enums) and includes the delivery handler which has no `@Plugin` representation.
**Trade-offs:** Dual representation — the `@Plugin` annotations and YAML both describe `iot.command` and `iot.state`. The YAML is the UI catalog source of truth; the `@Plugin` is the execution source of truth.
**Sources:** `fsitrading/app/src/main/resources/steps/trading-steps.yaml`, `io.casehub.pages.playbook.runtime.StepCatalogService`, `io.casehub.yaml.core.step.DeclarationParser`
**Exploration:** quick
**Status:** captured
