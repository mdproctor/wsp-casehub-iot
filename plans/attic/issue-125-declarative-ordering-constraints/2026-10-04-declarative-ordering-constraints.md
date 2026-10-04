# Declarative Ordering Constraints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #125 — feat: declarative ordering constraints for IoT desired state
**Issue group:** #125

**Goal:** Add DeviceClass-level ordering constraints to the IoT desired state system — declared in YAML, composed across presets and global files, enforced by the foundation planner, and visualized in the topology view.

**Architecture:** Make IoTNodeSpec implementations return composite NodeTypes (`"device-config/light"`, `"physical-device/lock"`) so the foundation `OrderingConstraint(NodeType, NodeType)` mechanism works at DeviceClass granularity. Add `ordering:` sections to preset YAML and global constraint files. The foundation `TransitionPlanner` handles all ordering enforcement — IoT only declares constraints and feeds them into the graph.

**Tech Stack:** Java 26, Quarkus 3.39, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- All SPIs are blocking (virtual-thread-aligned per ADR-0005)
- `casehub-iot-api` is a public API surface — no breaking changes without a major version bump
- `IoTNodeSpec` is a sealed interface — only `PhysicalDeviceSpec` and `DeviceConfigSpec` permitted
- Provider config properties must be `Optional<String>`
- Test assertions use AssertJ (`assertThat`, `assertThatThrownBy`)
- The `IoTGoalCompiler` calls `factory.of(nodes, deps)` — the 3-arg overload `factory.of(nodes, deps, orderingConstraints)` is available

---

## Batch 1: Foundation — NodeType utility and data model

### Task 1: IoTNodeTypes utility and IoTOrderingEntry record

**Files:**
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTNodeTypes.java`
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTOrderingEntry.java`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTNodeTypesTest.java`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTOrderingEntryTest.java`

**Interfaces:**
- Produces: `IoTNodeTypes.configType(DeviceClass)` → `NodeType`, `IoTNodeTypes.physicalType(DeviceClass)` → `NodeType`, `IoTNodeTypes.allConfig()` → `Set<NodeType>`, `IoTNodeTypes.allPhysical()` → `Set<NodeType>`, `IoTNodeTypes.all()` → `Set<NodeType>`, `IoTNodeTypes.extractDeviceClass(NodeType)` → `String`
- Produces: `IoTOrderingEntry(DeviceClass before, DeviceClass after)`

- [ ] **Step 1: Write failing tests for IoTNodeTypes**

```java
package io.casehub.iot.desiredstate;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.iot.api.DeviceClass;
import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class IoTNodeTypesTest {

    @Test
    void configType_returnsCompositeNodeType() {
        assertThat(IoTNodeTypes.configType(DeviceClass.LIGHT))
            .isEqualTo(NodeType.of("device-config/light"));
    }

    @Test
    void physicalType_returnsCompositeNodeType() {
        assertThat(IoTNodeTypes.physicalType(DeviceClass.LOCK))
            .isEqualTo(NodeType.of("physical-device/lock"));
    }

    @Test
    void allConfig_containsAllDeviceClasses() {
        Set<NodeType> all = IoTNodeTypes.allConfig();
        assertThat(all).hasSize(DeviceClass.values().length);
        assertThat(all).contains(NodeType.of("device-config/light"));
        assertThat(all).contains(NodeType.of("device-config/thermostat"));
    }

    @Test
    void allPhysical_containsAllDeviceClasses() {
        Set<NodeType> all = IoTNodeTypes.allPhysical();
        assertThat(all).hasSize(DeviceClass.values().length);
        assertThat(all).contains(NodeType.of("physical-device/switch"));
    }

    @Test
    void all_containsBothPrefixesPlusReview() {
        Set<NodeType> all = IoTNodeTypes.all();
        assertThat(all).hasSize(DeviceClass.values().length * 2 + 1);
        assertThat(all).contains(NodeType.of("iot-review"));
    }

    @Test
    void extractDeviceClass_fromConfigType() {
        assertThat(IoTNodeTypes.extractDeviceClass(NodeType.of("device-config/lock")))
            .isEqualTo("LOCK");
    }

    @Test
    void extractDeviceClass_fromPhysicalType() {
        assertThat(IoTNodeTypes.extractDeviceClass(NodeType.of("physical-device/thermostat")))
            .isEqualTo("THERMOSTAT");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl desiredstate test -Dtest=IoTNodeTypesTest -q`
Expected: compilation failure — `IoTNodeTypes` does not exist

- [ ] **Step 3: Implement IoTNodeTypes**

```java
package io.casehub.iot.desiredstate;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.iot.api.DeviceClass;

import java.util.HashSet;
import java.util.Set;

public final class IoTNodeTypes {

    public static final NodeType IOT_REVIEW = NodeType.of("iot-review");

    private static final Set<NodeType> ALL_CONFIG;
    private static final Set<NodeType> ALL_PHYSICAL;
    private static final Set<NodeType> ALL;

    static {
        var config = new HashSet<NodeType>();
        var physical = new HashSet<NodeType>();
        for (DeviceClass dc : DeviceClass.values()) {
            config.add(configType(dc));
            physical.add(physicalType(dc));
        }
        ALL_CONFIG = Set.copyOf(config);
        ALL_PHYSICAL = Set.copyOf(physical);
        var all = new HashSet<NodeType>();
        all.addAll(ALL_CONFIG);
        all.addAll(ALL_PHYSICAL);
        all.add(IOT_REVIEW);
        ALL = Set.copyOf(all);
    }

    public static NodeType configType(DeviceClass dc) {
        return NodeType.of("device-config/" + dc.name().toLowerCase());
    }

    public static NodeType physicalType(DeviceClass dc) {
        return NodeType.of("physical-device/" + dc.name().toLowerCase());
    }

    public static Set<NodeType> allConfig() { return ALL_CONFIG; }
    public static Set<NodeType> allPhysical() { return ALL_PHYSICAL; }
    public static Set<NodeType> all() { return ALL; }

    public static String extractDeviceClass(NodeType type) {
        String v = type.value();
        int slash = v.indexOf('/');
        if (slash < 0) throw new IllegalArgumentException("Not a composite NodeType: " + v);
        return v.substring(slash + 1).toUpperCase();
    }

    private IoTNodeTypes() {}
}
```

- [ ] **Step 4: Run IoTNodeTypesTest to verify it passes**

Run: `mvn -pl desiredstate test -Dtest=IoTNodeTypesTest -q`
Expected: all 7 tests pass

- [ ] **Step 5: Write failing tests for IoTOrderingEntry**

```java
package io.casehub.iot.desiredstate;

import io.casehub.iot.api.DeviceClass;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IoTOrderingEntryTest {

    @Test
    void validEntry() {
        var entry = new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT);
        assertThat(entry.before()).isEqualTo(DeviceClass.LOCK);
        assertThat(entry.after()).isEqualTo(DeviceClass.LIGHT);
    }

    @Test
    void nullBefore_throws() {
        assertThatThrownBy(() -> new IoTOrderingEntry(null, DeviceClass.LIGHT))
            .isInstanceOf(NullPointerException.class)
            .hasMessageContaining("before");
    }

    @Test
    void nullAfter_throws() {
        assertThatThrownBy(() -> new IoTOrderingEntry(DeviceClass.LOCK, null))
            .isInstanceOf(NullPointerException.class)
            .hasMessageContaining("after");
    }

    @Test
    void selfReference_throws() {
        assertThatThrownBy(() -> new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LOCK))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Self-referencing");
    }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `mvn -pl desiredstate test -Dtest=IoTOrderingEntryTest -q`
Expected: compilation failure — `IoTOrderingEntry` does not exist

- [ ] **Step 7: Implement IoTOrderingEntry**

```java
package io.casehub.iot.desiredstate;

import io.casehub.iot.api.DeviceClass;
import java.util.Objects;

public record IoTOrderingEntry(DeviceClass before, DeviceClass after) {
    public IoTOrderingEntry {
        Objects.requireNonNull(before, "ordering 'before' required");
        Objects.requireNonNull(after, "ordering 'after' required");
        if (before == after) {
            throw new IllegalArgumentException(
                "Self-referencing ordering constraint: " + before);
        }
    }
}
```

- [ ] **Step 8: Run IoTOrderingEntryTest to verify it passes**

Run: `mvn -pl desiredstate test -Dtest=IoTOrderingEntryTest -q`
Expected: all 4 tests pass

- [ ] **Step 9: Commit**

```bash
git add desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTNodeTypes.java desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTOrderingEntry.java desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTNodeTypesTest.java desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTOrderingEntryTest.java
git commit -m "feat(#125): add IoTNodeTypes utility and IoTOrderingEntry record"
```

### Task 2: Composite NodeTypes — update specs and consumers

**Files:**
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/DeviceConfigSpec.java:24`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/PhysicalDeviceSpec.java:17`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTNodeProvisioner.java:49-51`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTActualStateAdapter.java:28-29`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTFaultPolicy.java:22-23`
- Modify: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTGoalCompilerTest.java:60`
- Modify: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTNodeSpecTest.java`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTNodeTypesConsistencyTest.java`

**Interfaces:**
- Consumes: `IoTNodeTypes.configType(DeviceClass)`, `IoTNodeTypes.physicalType(DeviceClass)`, `IoTNodeTypes.all()`, `IoTNodeTypes.allConfig()`
- Produces: `DeviceConfigSpec.nodeType()` returns `NodeType.of("device-config/" + deviceClass.name().toLowerCase())`, `PhysicalDeviceSpec.nodeType()` returns `NodeType.of("physical-device/" + deviceClass.name().toLowerCase())`

- [ ] **Step 1: Write the consistency test (will fail until all changes are in)**

```java
package io.casehub.iot.desiredstate;

import io.casehub.desiredstate.api.NodeType;
import io.casehub.iot.api.DeviceClass;
import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class IoTNodeTypesConsistencyTest {

    @Test
    void provisioner_handlesAllDeviceClassVariants() {
        var provisioner = new IoTNodeProvisioner(null, java.util.List.of());
        Set<NodeType> handled = provisioner.handledTypes();
        for (DeviceClass dc : DeviceClass.values()) {
            assertThat(handled).contains(IoTNodeTypes.configType(dc));
            assertThat(handled).contains(IoTNodeTypes.physicalType(dc));
        }
        assertThat(handled).contains(IoTNodeTypes.IOT_REVIEW);
    }

    @Test
    void actualStateAdapter_handlesAllDeviceClassVariants() {
        var adapter = new IoTActualStateAdapter(null);
        Set<NodeType> handled = adapter.handledTypes();
        for (DeviceClass dc : DeviceClass.values()) {
            assertThat(handled).contains(IoTNodeTypes.configType(dc));
            assertThat(handled).contains(IoTNodeTypes.physicalType(dc));
        }
    }

    @Test
    void specTypes_matchNodeTypesUtility() {
        for (DeviceClass dc : DeviceClass.values()) {
            var configSpec = new DeviceConfigSpec("test", dc, java.util.Map.of());
            assertThat(configSpec.nodeType()).isEqualTo(IoTNodeTypes.configType(dc));

            var physicalSpec = new PhysicalDeviceSpec("test", dc, "Test");
            assertThat(physicalSpec.nodeType()).isEqualTo(IoTNodeTypes.physicalType(dc));
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl desiredstate test -Dtest=IoTNodeTypesConsistencyTest -q`
Expected: FAIL — NodeTypes don't match yet

- [ ] **Step 3: Update DeviceConfigSpec.nodeType()**

Change `desiredstate/src/main/java/io/casehub/iot/desiredstate/DeviceConfigSpec.java` line 24:

From: `public NodeType nodeType() { return NodeType.of("device-config"); }`
To: `public NodeType nodeType() { return IoTNodeTypes.configType(deviceClass); }`

- [ ] **Step 4: Update PhysicalDeviceSpec.nodeType()**

Change `desiredstate/src/main/java/io/casehub/iot/desiredstate/PhysicalDeviceSpec.java` line 17:

From: `public NodeType nodeType() { return NodeType.of("physical-device"); }`
To: `public NodeType nodeType() { return IoTNodeTypes.physicalType(deviceClass); }`

- [ ] **Step 5: Update IoTNodeProvisioner.handledTypes()**

Change `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTNodeProvisioner.java` line 49-51:

From:
```java
public Set<NodeType> handledTypes() {
    return Set.of(NodeType.of("physical-device"), NodeType.of("device-config"), NodeType.of("iot-review"));
}
```
To:
```java
public Set<NodeType> handledTypes() {
    return IoTNodeTypes.all();
}
```

- [ ] **Step 6: Update IoTActualStateAdapter.handledTypes()**

Change `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTActualStateAdapter.java` lines 28-29:

From:
```java
public Set<NodeType> handledTypes() {
    return Set.of(NodeType.of("physical-device"), NodeType.of("device-config"));
}
```
To:
```java
public Set<NodeType> handledTypes() {
    var types = new java.util.HashSet<>(IoTNodeTypes.allConfig());
    types.addAll(IoTNodeTypes.allPhysical());
    return Set.copyOf(types);
}
```

- [ ] **Step 7: Update IoTFaultPolicy nodeTypes**

Change `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTFaultPolicy.java` lines 22-28:

From:
```java
private static final NodeType DEVICE_CONFIG = NodeType.of("device-config");
private static final NodeType IOT_REVIEW    = NodeType.of("iot-review");

private final ThresholdFaultPolicy delegate = ThresholdFaultPolicy.builder()
    .faultTypes(Set.of(FaultType.PROVISION_FAILED))
    .nodeTypes(Set.of(DEVICE_CONFIG))
    .ignoreTypes(Set.of(IOT_REVIEW))
```
To:
```java
private final ThresholdFaultPolicy delegate = ThresholdFaultPolicy.builder()
    .faultTypes(Set.of(FaultType.PROVISION_FAILED))
    .nodeTypes(IoTNodeTypes.allConfig())
    .ignoreTypes(Set.of(IoTNodeTypes.IOT_REVIEW))
```

Remove the now-unused `DEVICE_CONFIG` and `IOT_REVIEW` static fields.

- [ ] **Step 8: Update existing test assertions**

In `IoTGoalCompilerTest.java` line 60, change:
```java
assertThat(node.type().value()).isEqualTo("device-config");
```
To:
```java
assertThat(node.type()).isEqualTo(IoTNodeTypes.configType(DeviceClass.LIGHT));
```

In `IoTNodeSpecTest.java` line 15, change:
`assertThat(spec.nodeType()).isEqualTo(NodeType.of("device-config"));`
To:
`assertThat(spec.nodeType()).isEqualTo(IoTNodeTypes.configType(DeviceClass.SWITCH));`

In `IoTNodeSpecTest.java` line 23, change:
`assertThat(spec.nodeType()).isEqualTo(NodeType.of("physical-device"));`
To:
`assertThat(spec.nodeType()).isEqualTo(IoTNodeTypes.physicalType(DeviceClass.THERMOSTAT));`

- [ ] **Step 9: Run all desiredstate tests**

Run: `mvn -pl desiredstate test -q`
Expected: all tests pass (including the new consistency test)

- [ ] **Step 10: Commit**

```bash
git add desiredstate/
git commit -m "feat(#125): switch to DeviceClass-aware composite NodeTypes"
```

---

## Batch 2: YAML and loading — ordering in goals and presets

### Task 3: Add ordering field to IoTGoals and update loaders

**Files:**
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoals.java`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalLoader.java:59-80,82-111`
- Modify: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTGoalLoaderTest.java`
- Create: `desiredstate/src/test/resources/iot-topology-with-ordering.yaml`
- Modify: `desiredstate/src/test/resources/presets/night-mode.yaml`

**Interfaces:**
- Consumes: `IoTOrderingEntry(DeviceClass, DeviceClass)`
- Produces: `IoTGoals.ordering()` → `List<IoTOrderingEntry>`, `IoTGoalLoader.merge()` and `mergeGoals()` union ordering entries

- [ ] **Step 1: Write failing test for YAML deserialization with ordering**

Add to `IoTGoalLoaderTest`:

```java
@Test
void load_withOrdering_deserializesEntries() {
    IoTGoals goals = loader.load("iot-topology-with-ordering.yaml");
    assertThat(goals.ordering()).hasSize(1);
    assertThat(goals.ordering().get(0).before()).isEqualTo(DeviceClass.LOCK);
    assertThat(goals.ordering().get(0).after()).isEqualTo(DeviceClass.LIGHT);
}

@Test
void load_withoutOrdering_defaultsToEmpty() {
    IoTGoals goals = loader.load("iot-topology-simple.yaml");
    assertThat(goals.ordering()).isEmpty();
}

@Test
void mergeGoals_unionsOrdering() {
    var a = new IoTGoals("t", List.of(
        new IoTDeviceGoal("d1", DeviceClass.SWITCH, "S", true, Map.of(), List.of())),
        List.of(new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT)));
    var b = new IoTGoals("t", List.of(
        new IoTDeviceGoal("d2", DeviceClass.LIGHT, "L", true, Map.of(), List.of())),
        List.of(new IoTOrderingEntry(DeviceClass.SWITCH, DeviceClass.COVER)));

    IoTGoals merged = IoTGoalLoader.mergeGoals(a, b);
    assertThat(merged.ordering()).hasSize(2);
    assertThat(merged.ordering()).contains(
        new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT),
        new IoTOrderingEntry(DeviceClass.SWITCH, DeviceClass.COVER));
}
```

- [ ] **Step 2: Create test YAML fixture**

Create `desiredstate/src/test/resources/iot-topology-with-ordering.yaml`:
```yaml
tenancyId: test-tenant
ordering:
  - before: LOCK
    after: LIGHT
devices:
  - deviceId: switch-1
    deviceClass: SWITCH
    label: Hallway Switch
    config:
      isOn: true
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl desiredstate test -Dtest=IoTGoalLoaderTest -q`
Expected: FAIL — `IoTGoals` constructor doesn't accept ordering parameter

- [ ] **Step 4: Update IoTGoals record**

Replace `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoals.java`:

```java
package io.casehub.iot.desiredstate;

import java.util.List;
import java.util.Objects;

public record IoTGoals(String tenancyId, List<IoTDeviceGoal> devices, List<IoTOrderingEntry> ordering) {
    public IoTGoals {
        Objects.requireNonNull(tenancyId, "tenancyId required");
        devices = List.copyOf(devices);
        ordering = ordering != null ? List.copyOf(ordering) : List.of();
    }

    public IoTGoals(String tenancyId, List<IoTDeviceGoal> devices) {
        this(tenancyId, devices, List.of());
    }
}
```

- [ ] **Step 5: Update IoTGoalLoader.merge() to union ordering**

In `IoTGoalLoader.java`, update the `merge()` method to collect ordering entries:

```java
public static IoTGoals merge(IoTGoals... fragments) {
    if (fragments.length == 0) throw new IllegalArgumentException("Cannot merge zero fragments");
    var seen = new HashSet<String>();
    var merged = new ArrayList<IoTDeviceGoal>();
    var allOrdering = new java.util.LinkedHashSet<IoTOrderingEntry>();
    String tenancyId = fragments[0].tenancyId();
    for (IoTGoals fragment : fragments) {
        if (!fragment.tenancyId().equals(tenancyId)) {
            throw new IllegalArgumentException("Inconsistent tenancyId in merge: expected " + tenancyId + ", found " + fragment.tenancyId());
        }
        for (IoTDeviceGoal device : fragment.devices()) {
            if (!seen.add(device.deviceId())) {
                throw new IllegalArgumentException("Duplicate deviceId in merge: " + device.deviceId());
            }
            merged.add(device);
        }
        allOrdering.addAll(fragment.ordering());
    }
    return new IoTGoals(tenancyId, merged, List.copyOf(allOrdering));
}
```

Do the same for `mergeGoals()`:

```java
public static IoTGoals mergeGoals(IoTGoals... fragments) {
    if (fragments.length == 0) throw new IllegalArgumentException("Cannot merge zero fragments");
    String tenancyId = fragments[0].tenancyId();
    var merged = new java.util.LinkedHashMap<String, IoTDeviceGoal>();
    var allOrdering = new java.util.LinkedHashSet<IoTOrderingEntry>();
    for (IoTGoals fragment : fragments) {
        if (!fragment.tenancyId().equals(tenancyId)) {
            throw new IllegalArgumentException("Inconsistent tenancyId in merge: expected " + tenancyId + ", found " + fragment.tenancyId());
        }
        for (IoTDeviceGoal device : fragment.devices()) {
            IoTDeviceGoal existing = merged.get(device.deviceId());
            if (existing != null) {
                var deepMerged = new java.util.HashMap<>(existing.config());
                deepMerged.putAll(device.config());
                merged.put(device.deviceId(), new IoTDeviceGoal(
                    device.deviceId(), device.deviceClass(), device.label(),
                    device.physical(), deepMerged,
                    device.dependsOn().isEmpty() ? existing.dependsOn() : device.dependsOn()));
            } else {
                merged.put(device.deviceId(), device);
            }
        }
        allOrdering.addAll(fragment.ordering());
    }
    return new IoTGoals(tenancyId, List.copyOf(merged.values()), List.copyOf(allOrdering));
}
```

- [ ] **Step 6: Run all desiredstate tests**

Run: `mvn -pl desiredstate test -q`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git add desiredstate/
git commit -m "feat(#125): add ordering field to IoTGoals with union merge semantics"
```

### Task 4: Global constraint loader and preset ordering propagation

**Files:**
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTOrderingConfig.java`
- Create: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTOrderingLoader.java`
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTPresetResolver.java:56-65`
- Test: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTOrderingLoaderTest.java`
- Modify: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTPresetResolverTest.java`
- Create: `desiredstate/src/test/resources/ordering/power-safety.yaml`
- Modify: `desiredstate/src/test/resources/presets/away-mode.yaml`

**Interfaces:**
- Consumes: `IoTOrderingEntry(DeviceClass, DeviceClass)`, `IoTGoals.ordering()`
- Produces: `IoTOrderingLoader.loadGlobal()` → `Set<IoTOrderingEntry>`, `IoTOrderingConfig.path()` → `Optional<String>`

- [ ] **Step 1: Write failing tests for IoTOrderingLoader**

```java
package io.casehub.iot.desiredstate;

import io.casehub.iot.api.DeviceClass;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class IoTOrderingLoaderTest {

    @TempDir Path tempDir;

    @Test
    void loadGlobal_scansDirectoryForOrderingEntries() throws IOException {
        Files.writeString(tempDir.resolve("safety.yaml"),
            "ordering:\n  - before: LOCK\n    after: LIGHT\n");
        var loader = new IoTOrderingLoader(tempDir.toString());
        Set<IoTOrderingEntry> entries = loader.loadGlobal();
        assertThat(entries).containsExactly(new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT));
    }

    @Test
    void loadGlobal_unionsMultipleFiles() throws IOException {
        Files.writeString(tempDir.resolve("a.yaml"),
            "ordering:\n  - before: LOCK\n    after: LIGHT\n");
        Files.writeString(tempDir.resolve("b.yaml"),
            "ordering:\n  - before: SWITCH\n    after: COVER\n");
        var loader = new IoTOrderingLoader(tempDir.toString());
        Set<IoTOrderingEntry> entries = loader.loadGlobal();
        assertThat(entries).hasSize(2);
    }

    @Test
    void loadGlobal_emptyDirectory_returnsEmpty() throws IOException {
        var loader = new IoTOrderingLoader(tempDir.toString());
        assertThat(loader.loadGlobal()).isEmpty();
    }

    @Test
    void loadGlobal_nullPath_returnsEmpty() {
        var loader = new IoTOrderingLoader(null);
        assertThat(loader.loadGlobal()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl desiredstate test -Dtest=IoTOrderingLoaderTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement IoTOrderingConfig**

```java
package io.casehub.iot.desiredstate;

import io.smallrye.config.ConfigMapping;
import java.util.Optional;

@ConfigMapping(prefix = "casehub.iot.ordering")
public interface IoTOrderingConfig {
    Optional<String> path();
}
```

- [ ] **Step 4: Implement IoTOrderingLoader**

```java
package io.casehub.iot.desiredstate;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.iot.api.DeviceClass;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.LinkedHashSet;
import java.util.Set;
import java.util.stream.Stream;

@ApplicationScoped
public class IoTOrderingLoader {

    private final String orderingDir;
    private final ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());

    @Inject
    public IoTOrderingLoader(IoTOrderingConfig config) {
        this.orderingDir = config.path().orElse(null);
    }

    public IoTOrderingLoader(String orderingDir) {
        this.orderingDir = orderingDir;
    }

    public Set<IoTOrderingEntry> loadGlobal() {
        if (orderingDir == null) return Set.of();
        Path dir = Path.of(orderingDir);
        if (!Files.isDirectory(dir)) return Set.of();
        var entries = new LinkedHashSet<IoTOrderingEntry>();
        try (Stream<Path> files = Files.list(dir)) {
            files.filter(this::isYaml).sorted().forEach(p -> entries.addAll(parseFile(p)));
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to list ordering directory", e);
        }
        return Set.copyOf(entries);
    }

    private Set<IoTOrderingEntry> parseFile(Path file) {
        try {
            JsonNode root = yamlMapper.readTree(file.toFile());
            JsonNode ordering = root.get("ordering");
            if (ordering == null || !ordering.isArray()) return Set.of();
            var entries = new LinkedHashSet<IoTOrderingEntry>();
            for (JsonNode entry : ordering) {
                DeviceClass before = DeviceClass.valueOf(entry.get("before").asText().toUpperCase());
                DeviceClass after = DeviceClass.valueOf(entry.get("after").asText().toUpperCase());
                entries.add(new IoTOrderingEntry(before, after));
            }
            return entries;
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to parse ordering file: " + file, e);
        }
    }

    private boolean isYaml(Path p) {
        String name = p.getFileName().toString();
        return name.endsWith(".yaml") || name.endsWith(".yml");
    }
}
```

- [ ] **Step 5: Run IoTOrderingLoaderTest to verify it passes**

Run: `mvn -pl desiredstate test -Dtest=IoTOrderingLoaderTest -q`
Expected: all 4 tests pass

- [ ] **Step 6: Write failing test for preset ordering propagation**

Add to `IoTPresetResolverTest`:

```java
@Test
void resolve_importsUnionOrdering() throws IOException {
    // night-mode.yaml has no ordering, away-mode imports night-mode + locks-armed
    // Add ordering to away-mode and verify it appears in resolved goals
    Path awayMode = Path.of(presetDir, "away-mode-ordered.yaml");
    Files.writeString(awayMode,
        "import:\n  - standalone\ntenancyId: test-tenant\nordering:\n  - before: LOCK\n    after: LIGHT\ndevices:\n  - deviceId: cam-1\n    deviceClass: CAMERA\n    label: Camera\n    config:\n      recording: true\n");

    IoTGoals goals = resolver.resolve("away-mode-ordered");
    assertThat(goals.ordering()).hasSize(1);
    assertThat(goals.ordering().get(0).before()).isEqualTo(DeviceClass.LOCK);
}
```

- [ ] **Step 7: Update IoTPresetResolver.loadPresetYaml() to preserve ordering**

The current `loadPresetYaml()` method strips the `import` key and deserializes via `loadFromNode()`. Since `IoTGoals` now has an `ordering` field, Jackson will deserialize it automatically. No code change needed for deserialization — only verify that `mergeGoals()` (already updated in Task 3) unions ordering entries.

Run test to verify: `mvn -pl desiredstate test -Dtest=IoTPresetResolverTest -q`

- [ ] **Step 8: Run all desiredstate tests**

Run: `mvn -pl desiredstate test -q`
Expected: all tests pass

- [ ] **Step 9: Commit**

```bash
git add desiredstate/
git commit -m "feat(#125): add IoTOrderingLoader for global constraints and preset ordering propagation"
```

---

## Batch 3: Compiler — wire constraints into graph with cycle detection

### Task 5: Update IoTGoalCompiler to resolve ordering constraints

**Files:**
- Modify: `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalCompiler.java`
- Modify: `desiredstate/src/test/java/io/casehub/iot/desiredstate/IoTGoalCompilerTest.java`

**Interfaces:**
- Consumes: `IoTOrderingEntry`, `IoTOrderingLoader.loadGlobal()`, `IoTNodeTypes.configType(DeviceClass)`, `IoTNodeTypes.physicalType(DeviceClass)`, `DesiredStateGraphFactory.of(nodes, deps, constraints)`
- Produces: Compiled `DesiredStateGraph` with `OrderingConstraint` records

- [ ] **Step 1: Write failing tests for constraint resolution**

Add to `IoTGoalCompilerTest`:

```java
@Test
void ordering_expandsToTwoOrderingConstraints() {
    var goals = new IoTGoals("tenant-1", List.of(
        new IoTDeviceGoal("lock-1", DeviceClass.LOCK, "Front Lock", false, Map.of("isLocked", true), List.of()),
        new IoTDeviceGoal("light-1", DeviceClass.LIGHT, "Hall Light", false, Map.of("isOn", true), List.of())),
        List.of(new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT)));

    DesiredStateGraph graph = ((CompilationResult.SingleGraph) compiler.compile(goals, factory)).graph();

    assertThat(graph.orderingConstraints()).hasSize(2);
    assertThat(graph.orderingConstraints()).contains(
        new OrderingConstraint(IoTNodeTypes.configType(DeviceClass.LOCK), IoTNodeTypes.configType(DeviceClass.LIGHT)));
    assertThat(graph.orderingConstraints()).contains(
        new OrderingConstraint(IoTNodeTypes.physicalType(DeviceClass.LOCK), IoTNodeTypes.physicalType(DeviceClass.LIGHT)));
}

@Test
void ordering_noEntries_noConstraints() {
    var goals = new IoTGoals("tenant-1", List.of(
        new IoTDeviceGoal("s-1", DeviceClass.SWITCH, "S", false, Map.of(), List.of())));

    DesiredStateGraph graph = ((CompilationResult.SingleGraph) compiler.compile(goals, factory)).graph();

    assertThat(graph.orderingConstraints()).isEmpty();
}

@Test
void ordering_cycleDetection_throwsAtCompileTime() {
    var goals = new IoTGoals("tenant-1", List.of(
        new IoTDeviceGoal("lock-1", DeviceClass.LOCK, "Lock", false, Map.of(), List.of()),
        new IoTDeviceGoal("light-1", DeviceClass.LIGHT, "Light", false, Map.of(), List.of())),
        List.of(
            new IoTOrderingEntry(DeviceClass.LOCK, DeviceClass.LIGHT),
            new IoTOrderingEntry(DeviceClass.LIGHT, DeviceClass.LOCK)));

    assertThatThrownBy(() -> compiler.compile(goals, factory))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("cycle");
}
```

Add imports: `import io.casehub.desiredstate.api.OrderingConstraint;`

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl desiredstate test -Dtest=IoTGoalCompilerTest -q`
Expected: FAIL — compiler doesn't handle ordering yet

- [ ] **Step 3: Update IoTGoalCompiler**

Replace `IoTGoalCompiler.java`:

```java
package io.casehub.iot.desiredstate;

import io.casehub.desiredstate.api.CompilationResult;
import io.casehub.desiredstate.api.Dependency;
import io.casehub.desiredstate.api.DesiredNode;
import io.casehub.desiredstate.api.DesiredStateGraphFactory;
import io.casehub.desiredstate.api.GoalCompiler;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.OrderingConstraint;
import io.casehub.iot.api.DeviceClass;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.ArrayDeque;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Queue;
import java.util.Set;

@ApplicationScoped
public class IoTGoalCompiler implements GoalCompiler<IoTGoals> {

    private final IoTOrderingLoader orderingLoader;

    @Inject
    public IoTGoalCompiler(IoTOrderingLoader orderingLoader) {
        this.orderingLoader = orderingLoader;
    }

    public IoTGoalCompiler() {
        this.orderingLoader = null;
    }

    @Override
    public CompilationResult compile(IoTGoals goals, DesiredStateGraphFactory factory) {
        Map<String, IoTDeviceGoal> lookup = new HashMap<>();
        for (IoTDeviceGoal goal : goals.devices()) {
            if (lookup.containsKey(goal.deviceId())) {
                throw new IllegalArgumentException("Duplicate deviceId: " + goal.deviceId());
            }
            lookup.put(goal.deviceId(), goal);
        }

        List<DesiredNode> nodes = new ArrayList<>();
        List<Dependency> deps = new ArrayList<>();

        for (IoTDeviceGoal goal : goals.devices()) {
            if (goal.physical()) {
                nodes.add(new DesiredNode(
                    NodeId.of(goal.deviceId()),
                    new PhysicalDeviceSpec(goal.deviceId(), goal.deviceClass(), goal.label()), HumanGating.ALL));
                nodes.add(new DesiredNode(
                    NodeId.of(goal.deviceId() + "-config"),
                    new DeviceConfigSpec(goal.deviceId(), goal.deviceClass(), goal.config()), HumanGating.NONE));
                deps.add(new Dependency(
                    NodeId.of(goal.deviceId() + "-config"), NodeId.of(goal.deviceId())));
            } else {
                nodes.add(new DesiredNode(
                    NodeId.of(goal.deviceId()),
                    new DeviceConfigSpec(goal.deviceId(), goal.deviceClass(), goal.config()), HumanGating.NONE));
            }

            for (String depId : goal.dependsOn()) {
                deps.add(new Dependency(NodeId.of(goal.deviceId()), NodeId.of(depId)));
            }
        }

        Set<OrderingConstraint> constraints = resolveConstraints(goals);
        return CompilationResult.single(factory.of(nodes, deps, constraints));
    }

    private Set<OrderingConstraint> resolveConstraints(IoTGoals goals) {
        var entries = new LinkedHashSet<IoTOrderingEntry>();
        if (orderingLoader != null) {
            entries.addAll(orderingLoader.loadGlobal());
        }
        entries.addAll(goals.ordering());

        if (entries.isEmpty()) return Set.of();

        validateNoCycles(entries);

        var constraints = new HashSet<OrderingConstraint>();
        for (IoTOrderingEntry entry : entries) {
            constraints.add(new OrderingConstraint(
                IoTNodeTypes.configType(entry.before()), IoTNodeTypes.configType(entry.after())));
            constraints.add(new OrderingConstraint(
                IoTNodeTypes.physicalType(entry.before()), IoTNodeTypes.physicalType(entry.after())));
        }
        return Set.copyOf(constraints);
    }

    private void validateNoCycles(Set<IoTOrderingEntry> entries) {
        Map<DeviceClass, Set<DeviceClass>> adj = new HashMap<>();
        Map<DeviceClass, Integer> inDegree = new HashMap<>();
        for (IoTOrderingEntry entry : entries) {
            adj.computeIfAbsent(entry.before(), k -> new HashSet<>()).add(entry.after());
            inDegree.putIfAbsent(entry.before(), 0);
            inDegree.merge(entry.after(), 1, Integer::sum);
        }
        Queue<DeviceClass> queue = new ArrayDeque<>();
        for (var e : inDegree.entrySet()) {
            if (e.getValue() == 0) queue.add(e.getKey());
        }
        int processed = 0;
        while (!queue.isEmpty()) {
            DeviceClass current = queue.poll();
            processed++;
            for (DeviceClass next : adj.getOrDefault(current, Set.of())) {
                int newDeg = inDegree.merge(next, -1, Integer::sum);
                if (newDeg == 0) queue.add(next);
            }
        }
        if (processed != inDegree.size()) {
            throw new IllegalArgumentException("Ordering constraints contain a cycle");
        }
    }
}
```

- [ ] **Step 4: Update IoTGoalCompilerTest setUp to use no-arg constructor**

The test's `setUp()` uses `new IoTGoalCompiler()` (no-arg). The new no-arg constructor sets `orderingLoader = null`, which is correct for tests that don't test global constraints.

- [ ] **Step 5: Run all desiredstate tests**

Run: `mvn -pl desiredstate test -q`
Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git add desiredstate/
git commit -m "feat(#125): wire ordering constraints through IoTGoalCompiler with early cycle detection"
```

---

## Batch 4: Topology — constraint visualization

### Task 6: Add ordering constraints to topology response

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/TopologyOrderingConstraint.java`
- Modify: `webapp-api/src/main/java/io/casehub/iot/webapp/rest/TopologyResponse.java`
- Modify: `webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyAssembler.java:76-86`
- Modify: `webapp/src/test/java/io/casehub/iot/webapp/app/service/TopologyAssemblerTest.java`

**Interfaces:**
- Consumes: `DesiredStateGraph.orderingConstraints()`, `IoTNodeTypes.extractDeviceClass(NodeType)`
- Produces: `TopologyResponse` with `List<TopologyOrderingConstraint> orderingConstraints`

- [ ] **Step 1: Write failing test for ordering constraints in topology**

Add to `TopologyAssemblerTest`:

```java
@Test
void assemble_includesOrderingConstraints() {
    // Set up a graph with ordering constraints and devices
    // Verify TopologyResponse.orderingConstraints() contains deduplicated class-level entries
    var graph = factory.of(
        List.of(
            new DesiredNode(NodeId.of("lock-1"),
                new DeviceConfigSpec("lock-1", DeviceClass.LOCK, Map.of()), HumanGating.NONE),
            new DesiredNode(NodeId.of("light-1"),
                new DeviceConfigSpec("light-1", DeviceClass.LIGHT, Map.of()), HumanGating.NONE)),
        List.of(),
        Set.of(
            new OrderingConstraint(IoTNodeTypes.configType(DeviceClass.LOCK), IoTNodeTypes.configType(DeviceClass.LIGHT)),
            new OrderingConstraint(IoTNodeTypes.physicalType(DeviceClass.LOCK), IoTNodeTypes.physicalType(DeviceClass.LIGHT))));

    // wire graph into assembler, call assemble
    TopologyResponse response = assembler.assemble("test-tenant");
    assertThat(response.orderingConstraints()).hasSize(1);
    assertThat(response.orderingConstraints().get(0).beforeClass()).isEqualTo("LOCK");
    assertThat(response.orderingConstraints().get(0).afterClass()).isEqualTo("LIGHT");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl webapp test -Dtest=TopologyAssemblerTest -q`
Expected: compilation failure

- [ ] **Step 3: Create TopologyOrderingConstraint**

```java
package io.casehub.iot.webapp.rest;

public record TopologyOrderingConstraint(String beforeClass, String afterClass) {}
```

- [ ] **Step 4: Update TopologyResponse**

```java
package io.casehub.iot.webapp.rest;

import java.util.List;
import java.util.Map;

public record TopologyResponse(
        List<TopologyNode> nodes,
        List<TopologyEdge> edges,
        List<TopologyOrderingConstraint> orderingConstraints,
        Map<String, TopologyAggregate> locationAggregates
) {}
```

- [ ] **Step 5: Update TopologyAssembler.assemble()**

After the dependency edge loop (line 86), add ordering constraint extraction:

```java
List<TopologyOrderingConstraint> orderingConstraints = new ArrayList<>();
if (graph != null) {
    var seen = new java.util.LinkedHashSet<String>();
    for (var c : graph.orderingConstraints()) {
        String before = IoTNodeTypes.extractDeviceClass(c.before());
        String after = IoTNodeTypes.extractDeviceClass(c.after());
        String key = before + "->" + after;
        if (seen.add(key)) {
            orderingConstraints.add(new TopologyOrderingConstraint(before, after));
        }
    }
}
```

Update the return statement to include `orderingConstraints`:
```java
return new TopologyResponse(List.copyOf(nodes), List.copyOf(edges),
    List.copyOf(orderingConstraints), Map.copyOf(aggregates));
```

Add import: `import io.casehub.iot.desiredstate.IoTNodeTypes;`
Add import: `import io.casehub.iot.webapp.rest.TopologyOrderingConstraint;`

- [ ] **Step 6: Fix all TopologyResponse constructor calls**

Search for all places that construct `TopologyResponse` — update to include the new `orderingConstraints` parameter. Use `List.of()` where no constraints are available.

- [ ] **Step 7: Run all webapp tests**

Run: `mvn -pl webapp test -q`
Expected: all tests pass

- [ ] **Step 8: Run full build**

Run: `mvn --batch-mode install -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add webapp-api/ webapp/
git commit -m "feat(#125): add ordering constraints to topology response"
```

## References

- [2026-10-04-declarative-ordering-constraints-design.md] — design spec this plan implements
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoalCompiler.java` — current compiler
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTGoals.java` — current goal record
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/IoTPresetResolver.java` — preset resolver
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/DeviceConfigSpec.java` — config spec
- `desiredstate/src/main/java/io/casehub/iot/desiredstate/PhysicalDeviceSpec.java` — physical spec
- `webapp/src/main/java/io/casehub/iot/webapp/app/service/TopologyAssembler.java` — topology assembly
- `io.casehub.desiredstate.api.OrderingConstraint` — foundation ordering constraint
- `io.casehub.desiredstate.runtime.TransitionPlanner` — foundation planner with virtual edge resolution
- GitHub #125 — focal issue
- GitHub #121 — parent epic
