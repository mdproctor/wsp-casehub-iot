# IoT Step Catalog Entries for yaml-core

**Issue:** casehubio/iot#134
**Branch:** issue-134-iot-step-catalog-entries
**Status:** Design

## Summary

Define YAML step catalog entries for the three IoT step types so the playbook library view and visual editor can offer them in the `PagesActionCatalog` UI. Follows the pattern established by fsitrading's `trading-steps.yaml`.

## Scope

**In scope:**
- YAML step definition file (`iot-steps.yaml`) with namespace `iot`
- Three catalog entries: `command`, `state`, `desired-state`
- Unit test validating the YAML parses correctly
- Wiring verification (step definitions directory on classpath)

**Out of scope:**
- Scenario CRUD REST API (#135)
- UI wiring of `PagesActionCatalog` component (separate work)
- Changes to the existing `@Plugin` Java implementations

## Architecture

### Step Catalog Data Flow

```
iot-steps.yaml                      PagesActionCatalog UI
  (classpath)                       (browser component)
      │                                    ▲
      ▼                                    │
StepCatalogService ──── GraphQL ───────────┘
  (Java, reads YAML)    /api/catalog/actions
```

The YAML file is the UI catalog source of truth. The `@Plugin` annotations remain the execution source of truth. Both describe the same step types — the YAML provides richer metadata (parameter descriptions, defaults, enums) for the catalog UI.

### File Location

`webapp/src/main/resources/steps/iot-steps.yaml`

Same location as fsitrading's `steps/trading-steps.yaml`. The `StepCatalogService` reads YAML files from a steps directory on the classpath.

## Catalog Entries

### 1. `command` (qualified: `iot.command`)

Dispatches a command to an IoT device via DeviceProvider.

**Inputs:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `device` | STRING | yes | Device identifier (e.g. `light-living-1`, `lock-front`) |
| `action` | STRING | yes | Command action (e.g. `turn_on`, `turn_off`, `lock`, `unlock`, `set_temperature`, `set_brightness`, `arm`, `disarm`) |
| `params` | OBJECT | no | Additional command parameters (e.g. `{ temperature: 22, unit: CELSIUS }`) |
| `correlation-id` | STRING | no | Correlation identifier for command tracking |

**Outputs:**

| Name | Type | Description |
|------|------|-------------|
| `result` | STRING | Command result: `SENT`, `FAILED`, or `TIMEOUT` |
| `device` | STRING | Device identifier echo |
| `action` | STRING | Action echo |
| `correlationId` | STRING | Correlation identifier used |

**Invoke:** `mcp: "iot_send_command"` — maps to the MCP tool surface for the catalog "Try It" panel.

### 2. `state` (qualified: `iot.state`)

Reads current state of an IoT device from DeviceRegistry.

**Inputs:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `device` | STRING | yes | Device identifier |

**Outputs:**

| Name | Type | Description |
|------|------|-------------|
| `deviceId` | STRING | Device identifier |
| `deviceClass` | STRING | Matter device class (e.g. `LIGHT`, `THERMOSTAT`, `LOCK`, `CONTACT_SENSOR`) |
| `providerId` | STRING | Provider identifier (e.g. `homeassistant`, `openhab`) |
| `capabilities` | OBJECT | Current device capabilities and state values |
| `location` | STRING | Hierarchical location path (nullable) |

**Invoke:** `mcp: "iot_get_state"` — maps to the MCP tool surface.

### 3. `desired-state` (qualified: `iot.desired-state`)

Applies a desired device state configuration via the desired-state convergence engine. Used with `delivery: desired-state` in playbooks. Accepts either a named preset or inline per-device goal specifications.

**Inputs:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `preset` | STRING | no | Named preset to apply (e.g. `nighttime`, `morning`, `away`). Resolved via `IoTPresetResolver` with import composition and last-wins merge. Mutually exclusive with inline device goals. |

When `preset` is not specified, remaining keys are treated as inline device goals — each key is a device ID mapping to a desired capability state object (e.g. `light-living-1: { state: "ON", brightness: 80 }`). This open-map pattern cannot be fully expressed in the catalog schema; the catalog entry documents the `preset` input and describes the inline alternative in the description.

**Outputs:**

| Name | Type | Description |
|------|------|-------------|
| `provisioned` | INTEGER | Number of devices successfully provisioned |
| `converged` | BOOLEAN | Whether all devices reached desired state |

**Invoke:** none (metadata-only). This is a delivery handler, not a step plugin — executed via `delivery: desired-state` syntax in playbooks, not as a catalog-resolvable action.

## Test Plan

`IoTStepDefinitionTest` in `webapp/src/test/java/io/casehub/iot/webapp/steps/`:

1. Load `steps/iot-steps.yaml` from classpath via `DeclarationParser.parse()`
2. Assert namespace is `iot`
3. Assert all three actions present: `command`, `state`, `desired-state`
4. Assert MCP invoke bindings for `command` and `state`
5. Assert `desired-state` has no invoke binding
6. Assert all actions have descriptions
7. Assert required/optional inputs are correctly typed

## References

- `fsitrading/app/src/main/resources/steps/trading-steps.yaml` — reference format (16 step definitions)
- `fsitrading/app/src/test/java/io/casehub/fsitrading/app/steps/TradingStepDefinitionTest.java` — reference test
- `io.casehub.yaml.core.step.DeclarationParser` — Java-side YAML parser
- `io.casehub.yaml.core.step.InvokeBinding` — sealed interface with invoke binding variants
- `io.casehub.pages.playbook.runtime.StepCatalogService` — server-side catalog loader
- `packages/pages-aria/src/controller/step-catalog.ts` — `PagesActionCatalog` UI component
- `docs/specs/issue-128-evaluate-playbook-ui/2026-10-07-playbook-ui-evaluation.md` — gap analysis identifying this work
- D1 in `decisions.md` — YAML catalog approach decision
