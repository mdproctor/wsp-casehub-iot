# Saved State Presets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #124 — saved state presets — named YAML configurations with composition
**Issue group:** #124

**Goal:** Add named desired state presets with YAML import/composition, last-wins merge, and a REST surface for listing, diffing, and applying presets.

**Architecture:** Presets are named `IoTGoals` YAML files in a configured directory. `IoTPresetResolver` handles name → path resolution and import expansion. A new `mergeGoals()` on `IoTGoalLoader` provides last-wins deep-merge (vs. existing `merge()` which rejects duplicates). The REST surface in webapp delegates to the resolver and existing desiredstate pipeline (compile → reconcile).

**Tech Stack:** Java 21, Quarkus, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- All new classes use `@ApplicationScoped` with blocking semantics (virtual threads, ADR-0005)
- Config properties use `Optional<String>` to avoid SmallRye startup failures
- REST endpoints follow `@McpDomain` pattern with `@ContextParam("tenancyId")`
- Package: `io.casehub.iot.desiredstate` for loader/resolver, `io.casehub.iot.webapp.app.service` for REST

---

## Batch 1: Core merge and resolution (desiredstate module)

### Task 1: IoTGoalLoader.mergeGoals() — last-wins deep merge

**Files:**
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalLoader.java:53-74`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTGoalLoaderTest.java`

**Interfaces:**
- Consumes: `IoTGoals`, `IoTDeviceGoal` records (existing)
- Produces: `IoTGoalLoader.mergeGoals(IoTGoals... fragments) → IoTGoals` — deep-merges config maps with last-wins semantics for overlapping deviceIds

- [ ] **Step 1: Write failing tests for mergeGoals()**

Add four tests to `IoTGoalLoaderTest`:

```java
@Test
void mergeGoals_nonOverlapping_combinesDevices() {
    var a = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.SWITCH, "Switch", true, Map.of("isOn", true), List.of())));
    var b = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-2", DeviceClass.LIGHT, "Light", true, Map.of("brightness", 80), List.of())));
    IoTGoals merged = IoTGoalLoader.mergeGoals(a, b);
    assertThat(merged.devices()).hasSize(2);
    assertThat(merged.tenancyId()).isEqualTo("t1");
}

@Test
void mergeGoals_overlappingDeviceId_deepMergesConfig() {
    var a = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.LIGHT, "Light", true, Map.of("isOn", true, "brightness", 30), List.of())));
    var b = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.LIGHT, "Light", true, Map.of("brightness", 80), List.of())));
    IoTGoals merged = IoTGoalLoader.mergeGoals(a, b);
    assertThat(merged.devices()).hasSize(1);
    var config = merged.devices().getFirst().config();
    assertThat(config).containsEntry("isOn", true);
    assertThat(config).containsEntry("brightness", 80);
}

@Test
void mergeGoals_overlappingDeviceId_laterWinsForNonConfigFields() {
    var a = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.LIGHT, "Light A", true, Map.of("isOn", true), List.of())));
    var b = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.LIGHT, "Light B", false, Map.of("brightness", 80), List.of())));
    IoTGoals merged = IoTGoalLoader.mergeGoals(a, b);
    assertThat(merged.devices().getFirst().label()).isEqualTo("Light B");
    assertThat(merged.devices().getFirst().physical()).isFalse();
}

@Test
void mergeGoals_inconsistentTenancyId_throws() {
    var a = new IoTGoals("t1", List.of(
        new IoTDeviceGoal("dev-1", DeviceClass.SWITCH, "Switch", true, Map.of("isOn", true), List.of())));
    var b = new IoTGoals("t2", List.of(
        new IoTDeviceGoal("dev-2", DeviceClass.LIGHT, "Light", true, Map.of("isOn", false), List.of())));
    assertThatThrownBy(() -> IoTGoalLoader.mergeGoals(a, b))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("Inconsistent tenancyId");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl desiredstate test -Dtest=IoTGoalLoaderTest`
Expected: FAIL — `mergeGoals` method does not exist

- [ ] **Step 3: Implement mergeGoals()**

Add to `IoTGoalLoader.java` after the existing `merge()` method (line 74):

```java
public static IoTGoals mergeGoals(IoTGoals... fragments) {
    if (fragments.length == 0) {
        throw new IllegalArgumentException("Cannot merge zero fragments");
    }
    String tenancyId = fragments[0].tenancyId();
    var merged = new java.util.LinkedHashMap<String, IoTDeviceGoal>();
    for (IoTGoals fragment : fragments) {
        if (!fragment.tenancyId().equals(tenancyId)) {
            throw new IllegalArgumentException(
                "Inconsistent tenancyId in merge: expected " + tenancyId + ", found " + fragment.tenancyId());
        }
        for (IoTDeviceGoal device : fragment.devices()) {
            IoTDeviceGoal existing = merged.get(device.deviceId());
            if (existing != null) {
                var deepMerged = new java.util.HashMap<>(existing.config());
                deepMerged.putAll(device.config());
                merged.put(device.deviceId(), new IoTDeviceGoal(
                    device.deviceId(),
                    device.deviceClass(),
                    device.label(),
                    device.physical(),
                    deepMerged,
                    device.dependsOn().isEmpty() ? existing.dependsOn() : device.dependsOn()));
            } else {
                merged.put(device.deviceId(), device);
            }
        }
    }
    return new IoTGoals(tenancyId, List.copyOf(merged.values()));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode -pl desiredstate test -Dtest=IoTGoalLoaderTest`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalLoader.java
git add desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTGoalLoaderTest.java
git commit -m "feat(#124): add mergeGoals() with last-wins deep merge"
```

### Task 2: IoTPresetConfig and IoTPresetResolver

**Files:**
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTPresetConfig.java`
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/PresetInfo.java`
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTPresetResolver.java`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTPresetResolverTest.java`
- Create: `desiredstate/src/test/resources/presets/night-mode.yaml`
- Create: `desiredstate/src/test/resources/presets/locks-armed.yaml`
- Create: `desiredstate/src/test/resources/presets/away-mode.yaml`
- Create: `desiredstate/src/test/resources/presets/standalone.yaml`

**Interfaces:**
- Consumes: `IoTGoalLoader` (existing), `IoTGoalLoader.mergeGoals()` (Task 1)
- Produces:
  - `IoTPresetConfig` — `@ConfigMapping(prefix = "casehub.iot.presets")` with `Optional<String> path()`
  - `PresetInfo` — `record PresetInfo(String name, List<String> imports, int deviceCount)`
  - `IoTPresetResolver.resolve(String name) → IoTGoals`
  - `IoTPresetResolver.listPresets() → List<PresetInfo>`

- [ ] **Step 1: Create test YAML fixtures**

`presets/night-mode.yaml`:
```yaml
tenancyId: test-tenant
devices:
  - deviceId: light-living
    deviceClass: LIGHT
    label: Living Room Light
    config:
      isOn: true
      brightness: 30
```

`presets/locks-armed.yaml`:
```yaml
tenancyId: test-tenant
devices:
  - deviceId: lock-front
    deviceClass: LOCK
    label: Front Door Lock
    config:
      locked: true
```

`presets/away-mode.yaml`:
```yaml
import:
  - night-mode
  - locks-armed
tenancyId: test-tenant
devices:
  - deviceId: light-living
    deviceClass: LIGHT
    label: Living Room Light
    config:
      brightness: 0
  - deviceId: camera-front
    deviceClass: CAMERA
    label: Front Camera
    config:
      recording: true
```

`presets/standalone.yaml`:
```yaml
tenancyId: test-tenant
devices:
  - deviceId: switch-hall
    deviceClass: SWITCH
    label: Hallway Switch
    config:
      isOn: false
```

- [ ] **Step 2: Write failing tests for IoTPresetResolver**

Create `IoTPresetResolverTest.java`:

```java
package io.casehub.iot.desiredstate;

import io.casehub.iot.api.DeviceClass;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.net.URISyntaxException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IoTPresetResolverTest {

    private IoTPresetResolver resolver;
    private String presetDir;

    @BeforeEach
    void setUp() throws URISyntaxException {
        var dirUrl = getClass().getClassLoader().getResource("presets");
        presetDir = Path.of(dirUrl.toURI()).toString();
        var loader = new IoTGoalLoader();
        resolver = new IoTPresetResolver(loader, presetDir);
    }

    @Test
    void resolve_standalonePreset_loadsDirectly() {
        IoTGoals goals = resolver.resolve("standalone");
        assertThat(goals.devices()).hasSize(1);
        assertThat(goals.devices().getFirst().deviceId()).isEqualTo("switch-hall");
    }

    @Test
    void resolve_presetWithImports_mergesInOrder() {
        IoTGoals goals = resolver.resolve("away-mode");
        assertThat(goals.tenancyId()).isEqualTo("test-tenant");
        var deviceIds = goals.devices().stream().map(IoTDeviceGoal::deviceId).toList();
        assertThat(deviceIds).containsExactlyInAnyOrder("light-living", "lock-front", "camera-front");
    }

    @Test
    void resolve_presetWithImports_lastWinsForOverlappingDevice() {
        IoTGoals goals = resolver.resolve("away-mode");
        var light = goals.devices().stream()
            .filter(d -> d.deviceId().equals("light-living")).findFirst().orElseThrow();
        assertThat(light.config()).containsEntry("brightness", 0);
        assertThat(light.config()).containsEntry("isOn", true);
    }

    @Test
    void resolve_missingPreset_throwsClearError() {
        assertThatThrownBy(() -> resolver.resolve("nonexistent"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("nonexistent");
    }

    @Test
    void resolve_missingImport_throwsClearError() throws IOException {
        Path tempDir = Files.createTempDirectory("presets");
        Files.writeString(tempDir.resolve("broken.yaml"),
            "import:\n  - does-not-exist\ntenancyId: t1\ndevices: []");
        var tempResolver = new IoTPresetResolver(new IoTGoalLoader(), tempDir.toString());
        assertThatThrownBy(() -> tempResolver.resolve("broken"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("does-not-exist");
    }

    @Test
    void listPresets_returnsAllPresetsWithMetadata() {
        List<PresetInfo> presets = resolver.listPresets();
        assertThat(presets).hasSizeGreaterThanOrEqualTo(4);
        var awayMode = presets.stream().filter(p -> p.name().equals("away-mode")).findFirst().orElseThrow();
        assertThat(awayMode.imports()).containsExactly("night-mode", "locks-armed");
        assertThat(awayMode.deviceCount()).isEqualTo(2);
    }

    @Test
    void listPresets_standaloneHasNoImports() {
        List<PresetInfo> presets = resolver.listPresets();
        var standalone = presets.stream().filter(p -> p.name().equals("standalone")).findFirst().orElseThrow();
        assertThat(standalone.imports()).isEmpty();
        assertThat(standalone.deviceCount()).isEqualTo(1);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode -pl desiredstate test -Dtest=IoTPresetResolverTest`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Create IoTPresetConfig**

```java
package io.casehub.iot.desiredstate;

import io.smallrye.config.ConfigMapping;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.iot.presets")
public interface IoTPresetConfig {
    Optional<String> path();
}
```

- [ ] **Step 5: Create PresetInfo record**

```java
package io.casehub.iot.desiredstate;

import java.util.List;

public record PresetInfo(String name, List<String> imports, int deviceCount) {
    public PresetInfo {
        imports = imports != null ? List.copyOf(imports) : List.of();
    }
}
```

- [ ] **Step 6: Implement IoTPresetResolver**

```java
package io.casehub.iot.desiredstate;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Stream;

@ApplicationScoped
public class IoTPresetResolver {

    private final IoTGoalLoader loader;
    private final String presetDir;
    private final ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());

    @Inject
    public IoTPresetResolver(IoTGoalLoader loader, IoTPresetConfig config) {
        this.loader = loader;
        this.presetDir = config.path().orElse(null);
    }

    // Test constructor
    IoTPresetResolver(IoTGoalLoader loader, String presetDir) {
        this.loader = loader;
        this.presetDir = presetDir;
    }

    public IoTGoals resolve(String name) {
        if (presetDir == null) {
            throw new IllegalStateException("casehub.iot.presets.path not configured");
        }
        Path presetPath = resolvePresetPath(name);
        List<String> imports = parseImports(presetPath);
        IoTGoals self = loader.load(presetPath.toString());

        if (imports.isEmpty()) {
            return self;
        }

        List<IoTGoals> fragments = new ArrayList<>();
        for (String importName : imports) {
            Path importPath = resolvePresetPath(importName);
            fragments.add(loader.load(importPath.toString()));
        }
        fragments.add(self);
        return IoTGoalLoader.mergeGoals(fragments.toArray(IoTGoals[]::new));
    }

    public List<PresetInfo> listPresets() {
        if (presetDir == null) {
            return List.of();
        }
        Path dir = Path.of(presetDir);
        if (!Files.isDirectory(dir)) {
            return List.of();
        }
        List<PresetInfo> result = new ArrayList<>();
        try (Stream<Path> files = Files.list(dir)) {
            files.filter(this::isYaml).sorted().forEach(p -> {
                String name = stripExtension(p.getFileName().toString());
                List<String> imports = parseImports(p);
                int deviceCount = countDevices(p);
                result.add(new PresetInfo(name, imports, deviceCount));
            });
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to list preset directory", e);
        }
        return result;
    }

    private Path resolvePresetPath(String name) {
        Path dir = Path.of(presetDir);
        Path yaml = dir.resolve(name + ".yaml");
        if (Files.exists(yaml)) return yaml;
        Path yml = dir.resolve(name + ".yml");
        if (Files.exists(yml)) return yml;
        throw new IllegalArgumentException("Preset not found: " + name
            + " (searched " + yaml + " and " + yml + ")");
    }

    private List<String> parseImports(Path presetPath) {
        try {
            JsonNode root = yamlMapper.readTree(presetPath.toFile());
            JsonNode importNode = root.get("import");
            if (importNode == null || !importNode.isArray()) {
                return List.of();
            }
            List<String> imports = new ArrayList<>();
            importNode.forEach(n -> imports.add(n.asText()));
            return imports;
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to parse preset: " + presetPath, e);
        }
    }

    private int countDevices(Path presetPath) {
        try {
            JsonNode root = yamlMapper.readTree(presetPath.toFile());
            JsonNode devices = root.get("devices");
            return devices != null && devices.isArray() ? devices.size() : 0;
        } catch (IOException e) {
            return 0;
        }
    }

    private boolean isYaml(Path p) {
        String name = p.getFileName().toString();
        return name.endsWith(".yaml") || name.endsWith(".yml");
    }

    private String stripExtension(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot > 0 ? filename.substring(0, dot) : filename;
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode -pl desiredstate test -Dtest=IoTPresetResolverTest`
Expected: PASS

- [ ] **Step 8: Run all desiredstate tests**

Run: `mvn --batch-mode -pl desiredstate test`
Expected: PASS — no regressions

- [ ] **Step 9: Commit**

```bash
git add desiredstate/
git commit -m "feat(#124): add IoTPresetResolver with config, name resolution, and import expansion"
```

## Batch 2: REST surface and diff (webapp module)

### Task 3: PresetDiff records and diff logic

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/rest/PresetDiff.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/PresetDiffCalculator.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/PresetDiffCalculatorTest.java`

**Interfaces:**
- Consumes: `IoTGoals`, `IoTDeviceGoal`, `DeviceConfigSpec` (desiredstate module), `DeviceEntity` and `DeviceRegistry` (api module)
- Produces:
  - `PresetDiff` — `record PresetDiff(List<DeviceChange> changes)`
  - `PresetDiff.DeviceChange` — `record DeviceChange(String deviceId, String deviceClass, List<PropertyChange> properties)`
  - `PresetDiff.PropertyChange` — `record PropertyChange(String property, Object current, Object desired)`
  - `PresetDiffCalculator.calculate(IoTGoals goals, DeviceRegistry registry, String tenancyId) → PresetDiff`

- [ ] **Step 1: Write failing tests for PresetDiffCalculator**

Create `PresetDiffCalculatorTest.java`:

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.DeviceClass;
import io.casehub.iot.api.SwitchDevice;
import io.casehub.iot.desiredstate.IoTDeviceGoal;
import io.casehub.iot.desiredstate.IoTGoals;
import io.casehub.iot.testing.MockDeviceRegistry;
import io.casehub.iot.webapp.rest.PresetDiff;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PresetDiffCalculatorTest {

    private MockDeviceRegistry registry;
    private PresetDiffCalculator calculator;

    @BeforeEach
    void setUp() {
        registry = new MockDeviceRegistry();
        calculator = new PresetDiffCalculator();
    }

    @Test
    void deviceWithChanges_showsPropertyDiff() {
        registry.addDevice(new SwitchDevice.Builder()
            .deviceId("switch-1").deviceClass(DeviceClass.SWITCH)
            .label("Switch").available(true).lastUpdated(Instant.EPOCH)
            .tenancyId("t1").providerId("test").on(false).build());

        var goals = new IoTGoals("t1", List.of(
            new IoTDeviceGoal("switch-1", DeviceClass.SWITCH, "Switch", true,
                Map.of("isOn", true), List.of())));

        PresetDiff diff = calculator.calculate(goals, registry, "t1");
        assertThat(diff.changes()).hasSize(1);
        assertThat(diff.changes().getFirst().deviceId()).isEqualTo("switch-1");
        var props = diff.changes().getFirst().properties();
        assertThat(props).anyMatch(p -> p.property().equals("isOn")
            && Boolean.FALSE.equals(p.current()) && Boolean.TRUE.equals(p.desired()));
    }

    @Test
    void deviceNotInRegistry_showsNullCurrent() {
        var goals = new IoTGoals("t1", List.of(
            new IoTDeviceGoal("new-device", DeviceClass.SWITCH, "New", true,
                Map.of("isOn", true), List.of())));

        PresetDiff diff = calculator.calculate(goals, registry, "t1");
        assertThat(diff.changes()).hasSize(1);
        var props = diff.changes().getFirst().properties();
        assertThat(props).anyMatch(p -> p.property().equals("isOn") && p.current() == null);
    }

    @Test
    void deviceAlreadyAtDesiredState_excludedFromDiff() {
        registry.addDevice(new SwitchDevice.Builder()
            .deviceId("switch-1").deviceClass(DeviceClass.SWITCH)
            .label("Switch").available(true).lastUpdated(Instant.EPOCH)
            .tenancyId("t1").providerId("test").on(true).build());

        var goals = new IoTGoals("t1", List.of(
            new IoTDeviceGoal("switch-1", DeviceClass.SWITCH, "Switch", true,
                Map.of("isOn", true), List.of())));

        PresetDiff diff = calculator.calculate(goals, registry, "t1");
        assertThat(diff.changes()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl webapp test -Dtest=PresetDiffCalculatorTest`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create PresetDiff record**

`webapp/src/main/java/io/casehub/iot/webapp/rest/PresetDiff.java`:

```java
package io.casehub.iot.webapp.rest;

import java.util.List;

public record PresetDiff(List<DeviceChange> changes) {
    public PresetDiff {
        changes = List.copyOf(changes);
    }

    public record DeviceChange(
        String deviceId,
        String deviceClass,
        List<PropertyChange> properties
    ) {
        public DeviceChange {
            properties = List.copyOf(properties);
        }
    }

    public record PropertyChange(String property, Object current, Object desired) {}
}
```

- [ ] **Step 4: Implement PresetDiffCalculator**

`webapp/src/main/java/io/casehub/iot/webapp/app/service/PresetDiffCalculator.java`:

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.desiredstate.IoTDeviceGoal;
import io.casehub.iot.desiredstate.IoTGoals;
import io.casehub.iot.webapp.rest.PresetDiff;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class PresetDiffCalculator {

    public PresetDiff calculate(IoTGoals goals, DeviceRegistry registry, String tenancyId) {
        List<PresetDiff.DeviceChange> changes = new ArrayList<>();

        for (IoTDeviceGoal goal : goals.devices()) {
            DeviceEntity device = registry.findAll().stream()
                .filter(d -> d.deviceId().equals(goal.deviceId()) && d.tenancyId().equals(tenancyId))
                .findFirst().orElse(null);

            Map<String, Object> currentState = device != null
                ? device.capabilities() : Map.of();

            List<PresetDiff.PropertyChange> propChanges = new ArrayList<>();
            for (var entry : goal.config().entrySet()) {
                Object current = currentState.get(entry.getKey());
                Object desired = entry.getValue();
                if (!java.util.Objects.equals(current, desired)) {
                    propChanges.add(new PresetDiff.PropertyChange(entry.getKey(), current, desired));
                }
            }

            if (!propChanges.isEmpty()) {
                changes.add(new PresetDiff.DeviceChange(
                    goal.deviceId(), goal.deviceClass().name(), propChanges));
            }
        }

        return new PresetDiff(changes);
    }
}
```

Note: `DeviceEntity.capabilities()` returns the device's current state as a `Map<String, Object>`. If this method doesn't exist, the implementer should check the actual `DeviceEntity` interface and use the appropriate method to extract current property values (e.g., iterating typed accessors or using a capabilities snapshot). Adjust the implementation to match the actual API.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -pl webapp test -Dtest=PresetDiffCalculatorTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add webapp/src/main/java/io/casehub/iot/webapp/rest/PresetDiff.java
git add webapp/src/main/java/io/casehub/iot/webapp/app/service/PresetDiffCalculator.java
git add webapp/src/test/java/io/casehub/iot/webapp/app/service/PresetDiffCalculatorTest.java
git commit -m "feat(#124): add PresetDiff records and PresetDiffCalculator"
```

### Task 4: DefaultIoTPresetApi REST surface

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/rest/PresetApplyResult.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTPresetApi.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTPresetApiTest.java`

**Interfaces:**
- Consumes: `IoTPresetResolver.resolve()`, `IoTPresetResolver.listPresets()` (Task 2), `IoTGoalCompiler` (existing), `PresetDiffCalculator.calculate()` (Task 3), `DeviceRegistry` (existing), `TransitionPlanner`, `IoTNodeProvisioner`, `IoTActualStateAdapter` (existing desiredstate)
- Produces:
  - `PresetApplyResult` — `record PresetApplyResult(String presetName, int provisioned, int skipped)`
  - `GET /iot/presets` → `List<PresetInfo>`
  - `GET /iot/presets/{name}/diff` → `PresetDiff`
  - `POST /iot/presets/{name}/apply` → `PresetApplyResult`

- [ ] **Step 1: Write failing tests for DefaultIoTPresetApi**

Create `DefaultIoTPresetApiTest.java`. Test the list and diff endpoints (apply requires more integration wiring — test via desiredstate runtime mock or integration test):

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.DeviceClass;
import io.casehub.iot.api.SwitchDevice;
import io.casehub.iot.desiredstate.IoTGoalLoader;
import io.casehub.iot.desiredstate.IoTPresetResolver;
import io.casehub.iot.desiredstate.PresetInfo;
import io.casehub.iot.testing.MockDeviceRegistry;
import io.casehub.iot.webapp.rest.PresetDiff;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.net.URISyntaxException;
import java.nio.file.Path;
import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class DefaultIoTPresetApiTest {

    private DefaultIoTPresetApi api;
    private MockDeviceRegistry registry;

    @BeforeEach
    void setUp() throws URISyntaxException {
        var dirUrl = getClass().getClassLoader().getResource("presets");
        String presetDir = Path.of(dirUrl.toURI()).toString();
        var loader = new IoTGoalLoader();
        var resolver = new IoTPresetResolver(loader, presetDir);
        registry = new MockDeviceRegistry();
        var diffCalculator = new PresetDiffCalculator();

        api = new DefaultIoTPresetApi();
        api.resolver = resolver;
        api.deviceRegistry = registry;
        api.diffCalculator = diffCalculator;
    }

    @Test
    void listPresets_returnsAllPresets() {
        List<PresetInfo> presets = api.listPresets("t1");
        assertThat(presets).isNotEmpty();
        assertThat(presets.stream().map(PresetInfo::name))
            .contains("night-mode", "locks-armed", "away-mode", "standalone");
    }

    @Test
    void diffPreset_showsExpectedChanges() {
        registry.addDevice(new SwitchDevice.Builder()
            .deviceId("switch-hall").deviceClass(DeviceClass.SWITCH)
            .label("Switch").available(true).lastUpdated(Instant.EPOCH)
            .tenancyId("test-tenant").providerId("test").on(true).build());

        PresetDiff diff = api.diffPreset("test-tenant", "standalone");
        assertThat(diff.changes()).hasSize(1);
        assertThat(diff.changes().getFirst().deviceId()).isEqualTo("switch-hall");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl webapp test -Dtest=DefaultIoTPresetApiTest`
Expected: FAIL — `DefaultIoTPresetApi` does not exist

- [ ] **Step 3: Create PresetApplyResult record**

`webapp/src/main/java/io/casehub/iot/webapp/rest/PresetApplyResult.java`:

```java
package io.casehub.iot.webapp.rest;

public record PresetApplyResult(String presetName, int provisioned, int skipped) {}
```

- [ ] **Step 4: Implement DefaultIoTPresetApi**

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.spi.DeviceRegistry;
import io.casehub.iot.desiredstate.IoTGoalCompiler;
import io.casehub.iot.desiredstate.IoTGoals;
import io.casehub.iot.desiredstate.IoTPresetResolver;
import io.casehub.iot.desiredstate.PresetInfo;
import io.casehub.iot.webapp.rest.PresetDiff;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import io.smallrye.common.annotation.Blocking;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;

@McpDomain(value = "iot/presets", app = "iot", basePath = "/api/presets",
    summary = "IoT saved state presets — list, preview, apply named configurations")
@ApplicationScoped
@Blocking
public class DefaultIoTPresetApi {

    @Inject IoTPresetResolver resolver;
    @Inject DeviceRegistry deviceRegistry;
    @Inject PresetDiffCalculator diffCalculator;

    @PlatformQuery("List available presets")
    @RestPath("/")
    public List<PresetInfo> listPresets(@ContextParam("tenancyId") String tenancyId) {
        return resolver.listPresets();
    }

    @PlatformQuery("Preview what changes a preset would make without applying it")
    @RestPath("/{name}/diff")
    public PresetDiff diffPreset(
            @ContextParam("tenancyId") String tenancyId,
            @PathParam("name") String name) {
        IoTGoals goals = resolver.resolve(name);
        return diffCalculator.calculate(goals, deviceRegistry, tenancyId);
    }

    @PlatformMutation("Apply a named preset — converges devices to the preset's desired state")
    @RestPath("/{name}/apply")
    public PresetApplyResult applyPreset(
            @ContextParam("tenancyId") String tenancyId,
            @PathParam("name") String name) {
        IoTGoals goals = resolver.resolve(name);
        DesiredStateGraph graph = ((CompilationResult.SingleGraph)
            compiler.compile(goals, graphFactory)).graph();
        ActualState actual = actualStateAdapter.readActual(graph, tenancyId);
        var plan = planner.plan(graph, actual);
        int provisioned = 0;
        int skipped = 0;
        for (var step : plan.flatAdditions()) {
            if (step.action() == StepAction.PROVISION) {
                var result = provisioner.provision(step.node(),
                    new ProvisionContext(tenancyId, graph));
                if (result instanceof ProvisionResult.Success) provisioned++;
                else skipped++;
            }
        }
        return new PresetApplyResult(name, provisioned, skipped);
    }
}
```

This requires additional injected fields:
```java
@Inject IoTGoalCompiler compiler;
@Inject DesiredStateGraphFactory graphFactory;
@Inject IoTActualStateAdapter actualStateAdapter;
@Inject TransitionPlanner planner;
@Inject IoTNodeProvisioner provisioner;
```

The pattern follows `IoTReconciliationIntegrationTest` — compile goals → read actual → plan transitions → provision each step.

- [ ] **Step 5: Copy test fixtures to webapp test resources**

The webapp tests need the same preset YAML fixtures. Copy the preset directory from `desiredstate/src/test/resources/presets/` to `webapp/src/test/resources/presets/`.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode -pl webapp test -Dtest=DefaultIoTPresetApiTest`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and tests pass

- [ ] **Step 8: Commit**

```bash
git add webapp/
git commit -m "feat(#124): add DefaultIoTPresetApi with list, diff, and apply endpoints"
```

## References

- [2026-10-03-saved-state-presets-design.md] — design spec this plan implements
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalLoader.java` — existing load/merge
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoals.java` — goal record
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTDeviceGoal.java` — device goal record
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalCompiler.java` — compilation
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/DeviceConfigSpec.java` — config spec
- `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTDeviceApi.java` — REST pattern
- Protocol: `harness-rest-resource-blocking-applicationscoped`
- Protocol: `configmapping-prefix-ownership`
- GitHub #124 — focal issue
- GitHub #121 — parent epic
