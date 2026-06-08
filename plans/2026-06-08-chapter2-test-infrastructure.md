# Chapter 2 — Test Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `capabilities()`, `toBuilder()`, and `deriveChangedCapabilities()` to the iot-api type hierarchy, then implement `MockDeviceProvider`, `MockDeviceRegistry`, `StateChangeEventPublisher`, and `Fixtures` in iot-testing.

**Architecture:** Two modules change. First, iot-api gains self-describing capabilities (each DeviceEntity subclass reports its capability map), copy-with-modify via `toBuilder()`, and a static utility to diff capability maps. Second, iot-testing gains four plain-Java/CDI classes for test use: a programmatic provider stub, a registry stub, a CDI event publisher, and static fixture factories. All changes are TDD — failing test first, minimal implementation second.

**Tech Stack:** Java 21, Quarkus ARC (CDI), JUnit 5, AssertJ, Maven multi-module (`mvn --batch-mode test -pl iot-api`, `mvn --batch-mode test -pl iot-testing`)

---

## File Map

**iot-api — modify:**
- `iot-api/src/main/java/io/casehub/iot/api/Temperature.java` — add scale-insensitive `equals()`/`hashCode()`
- `iot-api/src/main/java/io/casehub/iot/api/DeviceEntity.java` — add `CAP_AVAILABLE`, concrete `capabilities()`
- `iot-api/src/main/java/io/casehub/iot/api/SwitchDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/LightDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/ThermostatDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/SensorDevice.java` — remove `CAP_UNIT`, add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/PresenceSensor.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/PowerSensor.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/LockDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/CoverDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/MediaPlayerDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/FanDevice.java` — add `capabilities()`, `toBuilder()`
- `iot-api/src/main/java/io/casehub/iot/api/StateChangeEvent.java` — add `deriveChangedCapabilities()`
- `iot-api/src/test/java/io/casehub/iot/api/TemperatureTest.java` — add scale-sensitivity test
- `iot-api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java` — rewrite `stateChangeUpdatesRegistry()`
- `iot-api/src/test/java/io/casehub/iot/api/spi/DeviceRegistryContractTest.java` — **delete**

**iot-api — create:**
- `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`
- `iot-api/src/test/java/io/casehub/iot/api/DeriveChangedCapabilitiesTest.java`

**iot-testing — modify:**
- `iot-testing/pom.xml` — add `quarkus-junit` (test), `assertj-core` (test), Jandex plugin

**iot-testing — create:**
- `iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceProvider.java`
- `iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceRegistry.java`
- `iot-testing/src/main/java/io/casehub/iot/testing/StateChangeEventPublisher.java`
- `iot-testing/src/main/java/io/casehub/iot/testing/Fixtures.java`
- `iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceProviderTest.java`
- `iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceRegistryTest.java`
- `iot-testing/src/test/java/io/casehub/iot/testing/StateChangeEventPublisherTest.java`
- `iot-testing/src/test/java/io/casehub/iot/testing/FixturesTest.java`

---

## Task 1: Temperature.equals() — BigDecimal scale sensitivity fix

`BigDecimal.equals()` is scale-sensitive: `new BigDecimal("21") != new BigDecimal("21.0")`. The auto-generated record `equals()` delegates to this, causing `deriveChangedCapabilities()` to emit spurious capability changes when providers parse the same temperature with different decimal precision. Fix before any capability diffing can be trusted.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/Temperature.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/TemperatureTest.java`

- [ ] **Step 1: Add failing test to TemperatureTest.java**

Add this test method to the existing `TemperatureTest` class:

```java
@Test
void equalityIsScaleInsensitive() {
    var a = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
    var b = new Temperature(new BigDecimal("21.0"), Temperature.TemperatureUnit.CELSIUS);
    assertThat(a).isEqualTo(b);
    assertThat(a.hashCode()).isEqualTo(b.hashCode());
}

@Test
void differentScaleNotEqualToDifferentValue() {
    var a = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
    var b = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
    assertThat(a).isNotEqualTo(b);
}

@Test
void differentUnitNotEqual() {
    var a = new Temperature(new BigDecimal("100"), Temperature.TemperatureUnit.CELSIUS);
    var b = new Temperature(new BigDecimal("100"), Temperature.TemperatureUnit.FAHRENHEIT);
    assertThat(a).isNotEqualTo(b);
}
```

- [ ] **Step 2: Run and confirm failure**

```bash
mvn --batch-mode test -pl iot-api -Dtest=TemperatureTest#equalityIsScaleInsensitive
```

Expected: `FAIL — expected: 21 but was: 21.0` (or similar BigDecimal inequality message)

- [ ] **Step 3: Add custom equals/hashCode to Temperature.java**

Inside the record body, after the compact constructor, add:

```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof Temperature other)) return false;
    return unit == other.unit && value.compareTo(other.value) == 0;
}

@Override
public int hashCode() {
    return Objects.hash(unit, value.stripTrailingZeros());
}
```

`Objects` is already imported. No new imports needed.

- [ ] **Step 4: Run all Temperature tests**

```bash
mvn --batch-mode test -pl iot-api -Dtest=TemperatureTest
```

Expected: 7 tests, all PASS (the 4 original + 3 new).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/Temperature.java iot-api/src/test/java/io/casehub/iot/api/TemperatureTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "fix(iot-api): Temperature.equals() uses compareTo — scale-insensitive equality #2"
```

---

## Task 2: DeviceEntity.capabilities() base — CAP_AVAILABLE

`capabilities()` is a concrete method on `DeviceEntity` returning a mutable `LinkedHashMap`. It always contains `CAP_AVAILABLE`. Subclasses override with `super.capabilities()` + add their own fields. The mutable return is intentional — it enables the supplement chain (C3/C4) to add fields without re-allocating.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/DeviceEntity.java`
- Create: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java` (base tests only — subclass tests added in Tasks 3–9)

- [ ] **Step 1: Create CapabilitiesTest.java with base tests**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class CapabilitiesTest {

    static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    // Helper — minimal device for testing DeviceEntity base behavior
    private SwitchDevice sw(boolean on) {
        return SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(on).build();
    }

    @Test
    void baseCapabilitiesContainsCAPAVAILABLE() {
        var device = sw(false);
        assertThat(device.capabilities()).containsKey(DeviceEntity.CAP_AVAILABLE);
        assertThat(device.capabilities().get(DeviceEntity.CAP_AVAILABLE)).isEqualTo(true);
    }

    @Test
    void capabilitiesAllocatesFreshMapEachCall() {
        var device = sw(false);
        var caps1 = device.capabilities();
        caps1.put("injected", "value");
        assertThat(device.capabilities()).doesNotContainKey("injected");
    }

    @Test
    void unavailableDeviceCapabilitiesShowsFalse() {
        var device = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(false).lastUpdated(NOW).tenancyId("t1").on(false).build();
        assertThat(device.capabilities().get(DeviceEntity.CAP_AVAILABLE)).isEqualTo(false);
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-api -Dtest=CapabilitiesTest
```

Expected: COMPILATION ERROR — `DeviceEntity.CAP_AVAILABLE` does not exist.

- [ ] **Step 3: Add CAP_AVAILABLE constant and capabilities() to DeviceEntity.java**

Add imports at the top of DeviceEntity.java (after existing imports):
```java
import java.util.LinkedHashMap;
import java.util.Map;
```

Inside the `DeviceEntity` class, after the `tenancyId` field declaration, add:

```java
public static final String CAP_AVAILABLE = "available";

public Map<String, Object> capabilities() {
    Map<String, Object> caps = new LinkedHashMap<>();
    caps.put(CAP_AVAILABLE, available);
    return caps;
}
```

- [ ] **Step 4: Run base capability tests**

```bash
mvn --batch-mode test -pl iot-api -Dtest=CapabilitiesTest
```

Expected: 3 tests, all PASS. (SwitchDevice.capabilities() currently returns only {available} since SwitchDevice has not yet overridden it — that's correct for this task.)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/DeviceEntity.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): DeviceEntity.capabilities() base — CAP_AVAILABLE #2"
```

---

## Task 3: SwitchDevice — capabilities() + toBuilder()

Establishes the pattern for all subsequent device types. SwitchDevice uses a flat Builder (no AbstractBuilder) — it has no planned vendor supplement.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/SwitchDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java` (add SwitchDevice tests)
- Create: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java` (add SwitchDevice tests)

- [ ] **Step 1: Add SwitchDevice tests to CapabilitiesTest.java**

Add to the existing `CapabilitiesTest` class:

```java
@Test
void switchDeviceCapabilitiesContainsOnAndAvailable() {
    var device = SwitchDevice.builder()
        .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
        .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(SwitchDevice.CAP_ON, true);
    assertThat(caps).hasSize(2);
}

@Test
void switchDeviceCapabilitiesReflectsOffState() {
    var device = SwitchDevice.builder()
        .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
        .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
    assertThat(device.capabilities()).containsEntry(SwitchDevice.CAP_ON, false);
}
```

- [ ] **Step 2: Create ToBuilderTest.java with SwitchDevice tests**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class ToBuilderTest {

    static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @Test
    void switchDeviceToBuilderRoundTrip() {
        var original = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        var copy = original.toBuilder().build();
        assertThat(copy.deviceId()).isEqualTo("sw1");
        assertThat(copy.deviceClass()).isEqualTo(DeviceClass.SWITCH);
        assertThat(copy.label()).isEqualTo("Switch");
        assertThat(copy.available()).isTrue();
        assertThat(copy.lastUpdated()).isEqualTo(NOW);
        assertThat(copy.tenancyId()).isEqualTo("t1");
        assertThat(copy.isOn()).isTrue();
    }

    @Test
    void switchDeviceToBuilderModifyOn() {
        var original = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        SwitchDevice modified = original.toBuilder().on(false).build();
        assertThat(modified.isOn()).isFalse();
        assertThat(modified.deviceId()).isEqualTo("sw1");
        assertThat(modified).isInstanceOf(SwitchDevice.class);
    }
}
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: COMPILATION ERROR — `SwitchDevice` has no `toBuilder()`. The new capabilities tests fail because `capabilities()` returns only `{available}` (base class, not yet overridden).

- [ ] **Step 4: Add capabilities() and toBuilder() to SwitchDevice.java**

In SwitchDevice.java, add these methods inside the class body (after `isOn()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_ON, on);
    return caps;
}

public SwitchDevice.Builder toBuilder() {
    return SwitchDevice.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .on(on);
}
```

Add import at the top of SwitchDevice.java:
```java
import java.util.Map;
```

- [ ] **Step 5: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/SwitchDevice.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): SwitchDevice.capabilities() + toBuilder() #2"
```

---

## Task 4: LightDevice — capabilities() + toBuilder()

LightDevice uses AbstractBuilder for vendor supplement extension. `toBuilder()` returns `LightDevice.Builder` (the concrete inner class). Optional fields (`brightness`, `colorTemp`) are `Integer` — null means "not set, but the capability exists."

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/LightDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Add LightDevice tests to CapabilitiesTest.java**

```java
@Test
void lightDeviceCapabilitiesWithAllFields() {
    var device = LightDevice.builder()
        .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .on(true).brightness(200).colorTemp(4000).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(LightDevice.CAP_ON, true);
    assertThat(caps).containsEntry(LightDevice.CAP_BRIGHTNESS, 200);
    assertThat(caps).containsEntry(LightDevice.CAP_COLOR_TEMP, 4000);
    assertThat(caps).hasSize(4);
}

@Test
void lightDeviceCapabilitiesNullOptionalFieldsIncludedAsNull() {
    var device = LightDevice.builder()
        .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
        .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
    var caps = device.capabilities();
    assertThat(caps).containsKey(LightDevice.CAP_BRIGHTNESS);
    assertThat(caps.get(LightDevice.CAP_BRIGHTNESS)).isNull();
    assertThat(caps).containsKey(LightDevice.CAP_COLOR_TEMP);
    assertThat(caps.get(LightDevice.CAP_COLOR_TEMP)).isNull();
    assertThat(caps).hasSize(4);
}
```

- [ ] **Step 2: Add LightDevice tests to ToBuilderTest.java**

```java
@Test
void lightDeviceToBuilderRoundTrip() {
    var original = LightDevice.builder()
        .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .on(true).brightness(200).colorTemp(4000).build();
    var copy = original.toBuilder().build();
    assertThat(copy.deviceId()).isEqualTo("l1");
    assertThat(copy.isOn()).isTrue();
    assertThat(copy.brightness()).hasValue(200);
    assertThat(copy.colorTemp()).hasValue(4000);
    assertThat(copy).isInstanceOf(LightDevice.class);
}

@Test
void lightDeviceToBuilderModifyBrightness() {
    var original = LightDevice.builder()
        .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .on(true).brightness(200).build();
    LightDevice modified = original.toBuilder().brightness(100).build();
    assertThat(modified.brightness()).hasValue(100);
    assertThat(modified.isOn()).isTrue();
    assertThat(modified).isInstanceOf(LightDevice.class);
}

@Test
void lightDeviceToBuilderPreservesNullOptionals() {
    var original = LightDevice.builder()
        .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
        .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
    var copy = original.toBuilder().build();
    assertThat(copy.brightness()).isEmpty();
    assertThat(copy.colorTemp()).isEmpty();
}
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: LightDevice tests FAIL — `capabilities()` returns wrong keys, no `toBuilder()`.

- [ ] **Step 4: Add capabilities() and toBuilder() to LightDevice.java**

Add import at top of LightDevice.java:
```java
import java.util.Map;
```

Inside LightDevice class (after `colorTemp()` method):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_ON, on);
    caps.put(CAP_BRIGHTNESS, brightness);
    caps.put(CAP_COLOR_TEMP, colorTemp);
    return caps;
}

public LightDevice.Builder toBuilder() {
    return new Builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .on(on).brightness(brightness).colorTemp(colorTemp);
}
```

Note: `on`, `brightness`, `colorTemp` are `private final` fields of `LightDevice`, accessed directly inside the class.

- [ ] **Step 5: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/LightDevice.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): LightDevice.capabilities() + toBuilder() #2"
```

---

## Task 5: ThermostatDevice — capabilities() + toBuilder()

Temperature objects in the capabilities map are compared with `Objects.equals()` — which now works correctly thanks to Task 1. ThermostatDevice uses AbstractBuilder.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/ThermostatDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Add ThermostatDevice tests to CapabilitiesTest.java**

```java
@Test
void thermostatDeviceCapabilities() {
    var current = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
    var target = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
    var device = ThermostatDevice.builder()
        .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("Thermostat")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .currentTemperature(current).targetTemperature(target).mode(ThermostatMode.HEAT).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(ThermostatDevice.CAP_CURRENT_TEMPERATURE, current);
    assertThat(caps).containsEntry(ThermostatDevice.CAP_TARGET_TEMPERATURE, target);
    assertThat(caps).containsEntry(ThermostatDevice.CAP_MODE, ThermostatMode.HEAT);
    assertThat(caps).hasSize(4);
}
```

Add import at top of CapabilitiesTest.java:
```java
import java.math.BigDecimal;
```

- [ ] **Step 2: Add ThermostatDevice tests to ToBuilderTest.java**

```java
@Test
void thermostatDeviceToBuilderRoundTrip() {
    var current = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
    var target = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
    var original = ThermostatDevice.builder()
        .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("Thermostat")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .currentTemperature(current).targetTemperature(target).mode(ThermostatMode.HEAT).build();
    var copy = original.toBuilder().build();
    assertThat(copy.currentTemperature()).isEqualTo(current);
    assertThat(copy.targetTemperature()).isEqualTo(target);
    assertThat(copy.mode()).isEqualTo(ThermostatMode.HEAT);
    assertThat(copy).isInstanceOf(ThermostatDevice.class);
}

@Test
void thermostatDeviceToBuilderModifyTarget() {
    var current = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
    var target = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
    var newTarget = new Temperature(new BigDecimal("23"), Temperature.TemperatureUnit.CELSIUS);
    var original = ThermostatDevice.builder()
        .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("Thermostat")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .currentTemperature(current).targetTemperature(target).mode(ThermostatMode.HEAT).build();
    ThermostatDevice modified = original.toBuilder().targetTemperature(newTarget).build();
    assertThat(modified.targetTemperature()).isEqualTo(newTarget);
    assertThat(modified.currentTemperature()).isEqualTo(current);
}
```

Add import to ToBuilderTest.java:
```java
import java.math.BigDecimal;
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: ThermostatDevice tests FAIL.

- [ ] **Step 4: Add capabilities() and toBuilder() to ThermostatDevice.java**

Add import:
```java
import java.util.Map;
```

Inside ThermostatDevice class:

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_CURRENT_TEMPERATURE, currentTemperature);
    caps.put(CAP_TARGET_TEMPERATURE, targetTemperature);
    caps.put(CAP_MODE, mode);
    return caps;
}

public ThermostatDevice.Builder toBuilder() {
    return new Builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .currentTemperature(currentTemperature)
        .targetTemperature(targetTemperature)
        .mode(mode);
}
```

- [ ] **Step 5: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/ThermostatDevice.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): ThermostatDevice.capabilities() + toBuilder() #2"
```

---

## Task 6: SensorDevice — remove CAP_UNIT, add capabilities() + toBuilder()

`CAP_UNIT` is a public constant that misleads readers — `unit` is static sensor configuration, not a runtime capability. Delete it. `unit` remains a field and is preserved in `toBuilder()`. Check `SensorDeviceTest.java` for any references to `CAP_UNIT` and remove them too.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/SensorDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/SensorDeviceTest.java` (check and remove CAP_UNIT refs)
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Check SensorDeviceTest.java for CAP_UNIT references**

Open `iot-api/src/test/java/io/casehub/iot/api/SensorDeviceTest.java`. If it references `SensorDevice.CAP_UNIT`, remove those assertions or update them. `CAP_NUMERIC_VALUE` and `CAP_BINARY_VALUE` remain valid.

- [ ] **Step 2: Add SensorDevice tests to CapabilitiesTest.java**

```java
@Test
void sensorDeviceCapabilitiesExcludesUnit() {
    var device = SensorDevice.builder()
        .deviceId("s1").deviceClass(DeviceClass.SENSOR).label("Sensor")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .sensorType(SensorType.TEMPERATURE)
        .numericValue(new BigDecimal("21.5")).unit("C").build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsKey(SensorDevice.CAP_NUMERIC_VALUE);
    assertThat(caps).containsKey(SensorDevice.CAP_BINARY_VALUE);
    assertThat(caps).doesNotContainKey("unit");
    assertThat(caps).hasSize(3);
}

@Test
void sensorDeviceCapabilitiesNullValues() {
    var device = SensorDevice.builder()
        .deviceId("s1").deviceClass(DeviceClass.SENSOR).label("Sensor")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .sensorType(SensorType.MOTION).build();
    var caps = device.capabilities();
    assertThat(caps.get(SensorDevice.CAP_NUMERIC_VALUE)).isNull();
    assertThat(caps.get(SensorDevice.CAP_BINARY_VALUE)).isNull();
}
```

- [ ] **Step 3: Add SensorDevice tests to ToBuilderTest.java**

```java
@Test
void sensorDeviceToBuilderPreservesUnit() {
    var original = SensorDevice.builder()
        .deviceId("s1").deviceClass(DeviceClass.SENSOR).label("Sensor")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .sensorType(SensorType.TEMPERATURE)
        .numericValue(new BigDecimal("21.5")).unit("C").build();
    var copy = original.toBuilder().build();
    assertThat(copy.unit()).hasValue("C");
    assertThat(copy.numericValue()).hasValue(new BigDecimal("21.5"));
    assertThat(copy.sensorType()).isEqualTo(SensorType.TEMPERATURE);
}

@Test
void sensorDeviceToBuilderModifyNumericValue() {
    var original = SensorDevice.builder()
        .deviceId("s1").deviceClass(DeviceClass.SENSOR).label("Sensor")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .sensorType(SensorType.TEMPERATURE)
        .numericValue(new BigDecimal("21")).unit("C").build();
    SensorDevice modified = original.toBuilder().numericValue(new BigDecimal("22")).build();
    assertThat(modified.numericValue()).hasValue(new BigDecimal("22"));
    assertThat(modified.unit()).hasValue("C");
}
```

- [ ] **Step 4: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest+SensorDeviceTest"
```

Expected: COMPILATION ERRORS — `CAP_UNIT` referenced in SensorDeviceTest (if any) and missing methods.

- [ ] **Step 5: Update SensorDevice.java — delete CAP_UNIT, add capabilities() and toBuilder()**

In SensorDevice.java:

1. **Delete** the line: `public static final String CAP_UNIT = "unit";`

2. Add import:
```java
import java.util.Map;
```

3. Add these methods inside the class body (after `binaryValue()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_NUMERIC_VALUE, numericValue);
    caps.put(CAP_BINARY_VALUE, binaryValue);
    return caps;
}

public SensorDevice.Builder toBuilder() {
    return SensorDevice.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .sensorType(sensorType).numericValue(numericValue).unit(unit).binaryValue(binaryValue);
}
```

- [ ] **Step 6: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest+SensorDeviceTest"
```

Expected: All tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/SensorDevice.java iot-api/src/test/java/io/casehub/iot/api/SensorDeviceTest.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): SensorDevice — remove CAP_UNIT, add capabilities() + toBuilder() #2"
```

---

## Task 7: PresenceSensor + PowerSensor — capabilities() + toBuilder()

Both use flat Builder (no supplement planned).

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/PresenceSensor.java`
- Modify: `iot-api/src/main/java/io/casehub/iot/api/PowerSensor.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Add tests to CapabilitiesTest.java**

```java
@Test
void presenceSensorCapabilities() {
    var device = PresenceSensor.builder()
        .deviceId("p1").deviceClass(DeviceClass.PRESENCE_SENSOR).label("Presence")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .present(true).lastSeen(NOW).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(PresenceSensor.CAP_PRESENT, true);
    assertThat(caps).containsEntry(PresenceSensor.CAP_LAST_SEEN, NOW);
    assertThat(caps).hasSize(3);
}

@Test
void powerSensorCapabilities() {
    var device = PowerSensor.builder()
        .deviceId("ps1").deviceClass(DeviceClass.POWER_SENSOR).label("Power")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .power(new BigDecimal("100")).energy(new BigDecimal("50")).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(PowerSensor.CAP_POWER, new BigDecimal("100"));
    assertThat(caps).containsEntry(PowerSensor.CAP_ENERGY, new BigDecimal("50"));
    assertThat(caps).hasSize(3);
}
```

Add import to CapabilitiesTest.java if not already present:
```java
import java.time.Instant;
```

- [ ] **Step 2: Add tests to ToBuilderTest.java**

```java
@Test
void presenceSensorToBuilderRoundTrip() {
    var original = PresenceSensor.builder()
        .deviceId("p1").deviceClass(DeviceClass.PRESENCE_SENSOR).label("Presence")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .present(false).lastSeen(NOW).build();
    var copy = original.toBuilder().build();
    assertThat(copy.isPresent()).isFalse();
    assertThat(copy.lastSeen()).isEqualTo(NOW);
    assertThat(copy).isInstanceOf(PresenceSensor.class);
}

@Test
void powerSensorToBuilderRoundTrip() {
    var original = PowerSensor.builder()
        .deviceId("ps1").deviceClass(DeviceClass.POWER_SENSOR).label("Power")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .power(new BigDecimal("100")).energy(new BigDecimal("50")).build();
    var copy = original.toBuilder().build();
    assertThat(copy.power()).isEqualByComparingTo(new BigDecimal("100"));
    assertThat(copy.energy()).isEqualByComparingTo(new BigDecimal("50"));
    assertThat(copy).isInstanceOf(PowerSensor.class);
}
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: PresenceSensor and PowerSensor tests FAIL.

- [ ] **Step 4: Add capabilities() and toBuilder() to PresenceSensor.java**

Add import:
```java
import java.util.Map;
```

Inside PresenceSensor class (after `lastSeen()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_PRESENT, present);
    caps.put(CAP_LAST_SEEN, lastSeen);
    return caps;
}

public PresenceSensor.Builder toBuilder() {
    return PresenceSensor.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .present(present).lastSeen(lastSeen);
}
```

- [ ] **Step 5: Add capabilities() and toBuilder() to PowerSensor.java**

Add import:
```java
import java.util.Map;
```

Inside PowerSensor class (after `energy()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_POWER, power);
    caps.put(CAP_ENERGY, energy);
    return caps;
}

public PowerSensor.Builder toBuilder() {
    return PowerSensor.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .power(power).energy(energy);
}
```

- [ ] **Step 6: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: All tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/PresenceSensor.java iot-api/src/main/java/io/casehub/iot/api/PowerSensor.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): PresenceSensor + PowerSensor capabilities() + toBuilder() #2"
```

---

## Task 8: LockDevice + CoverDevice — capabilities() + toBuilder()

Both use AbstractBuilder (vendor supplements planned: HomeAssistantLock, OpenHABRollershutter). `toBuilder()` uses `new Builder()` — the concrete inner Builder class.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/LockDevice.java`
- Modify: `iot-api/src/main/java/io/casehub/iot/api/CoverDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Add tests to CapabilitiesTest.java**

```java
@Test
void lockDeviceCapabilities() {
    var device = LockDevice.builder()
        .deviceId("lk1").deviceClass(DeviceClass.LOCK).label("Lock")
        .available(true).lastUpdated(NOW).tenancyId("t1").locked(true).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(LockDevice.CAP_LOCKED, true);
    assertThat(caps).hasSize(2);
}

@Test
void coverDeviceCapabilities() {
    var device = CoverDevice.builder()
        .deviceId("cv1").deviceClass(DeviceClass.COVER).label("Cover")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .position(75).moving(false).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(CoverDevice.CAP_POSITION, 75);
    assertThat(caps).containsEntry(CoverDevice.CAP_MOVING, false);
    assertThat(caps).hasSize(3);
}
```

- [ ] **Step 2: Add tests to ToBuilderTest.java**

```java
@Test
void lockDeviceToBuilderModifyLocked() {
    var original = LockDevice.builder()
        .deviceId("lk1").deviceClass(DeviceClass.LOCK).label("Lock")
        .available(true).lastUpdated(NOW).tenancyId("t1").locked(true).build();
    LockDevice unlocked = original.toBuilder().locked(false).build();
    assertThat(unlocked.isLocked()).isFalse();
    assertThat(unlocked.deviceId()).isEqualTo("lk1");
    assertThat(unlocked).isInstanceOf(LockDevice.class);
}

@Test
void coverDeviceToBuilderModifyPosition() {
    var original = CoverDevice.builder()
        .deviceId("cv1").deviceClass(DeviceClass.COVER).label("Cover")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .position(0).moving(false).build();
    CoverDevice opened = original.toBuilder().position(100).build();
    assertThat(opened.position()).isEqualTo(100);
    assertThat(opened.isMoving()).isFalse();
    assertThat(opened).isInstanceOf(CoverDevice.class);
}
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: LockDevice and CoverDevice tests FAIL.

- [ ] **Step 4: Add capabilities() and toBuilder() to LockDevice.java**

Add import:
```java
import java.util.Map;
```

Inside `LockDevice` class (in the non-abstract part, after `isLocked()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_LOCKED, locked);
    return caps;
}

public LockDevice.Builder toBuilder() {
    return new Builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .locked(locked);
}
```

Note: `locked` is a field in `LockDevice.AbstractBuilder` but stored in `LockDevice` — it is accessible as `locked` directly inside the class.

- [ ] **Step 5: Add capabilities() and toBuilder() to CoverDevice.java**

Add import:
```java
import java.util.Map;
```

Inside `CoverDevice` class (after `isMoving()`):

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_POSITION, position);
    caps.put(CAP_MOVING, moving);
    return caps;
}

public CoverDevice.Builder toBuilder() {
    return new Builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .position(position).moving(moving);
}
```

- [ ] **Step 6: Run and confirm pass**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: All tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/LockDevice.java iot-api/src/main/java/io/casehub/iot/api/CoverDevice.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): LockDevice + CoverDevice capabilities() + toBuilder() #2"
```

---

## Task 9: MediaPlayerDevice + FanDevice — capabilities() + toBuilder()

Both use flat Builder. Both have optional numeric fields (`Integer volume`, `Integer speed`).

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/MediaPlayerDevice.java`
- Modify: `iot-api/src/main/java/io/casehub/iot/api/FanDevice.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java`
- Modify: `iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java`

- [ ] **Step 1: Add tests to CapabilitiesTest.java**

```java
@Test
void mediaPlayerDeviceCapabilities() {
    var device = MediaPlayerDevice.builder()
        .deviceId("mp1").deviceClass(DeviceClass.MEDIA_PLAYER).label("Player")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .playing(true).volume(80).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(MediaPlayerDevice.CAP_PLAYING, true);
    assertThat(caps).containsEntry(MediaPlayerDevice.CAP_VOLUME, 80);
    assertThat(caps).hasSize(3);
}

@Test
void mediaPlayerNullVolumeIncludedAsNull() {
    var device = MediaPlayerDevice.builder()
        .deviceId("mp1").deviceClass(DeviceClass.MEDIA_PLAYER).label("Player")
        .available(true).lastUpdated(NOW).tenancyId("t1").playing(false).build();
    assertThat(device.capabilities()).containsKey(MediaPlayerDevice.CAP_VOLUME);
    assertThat(device.capabilities().get(MediaPlayerDevice.CAP_VOLUME)).isNull();
}

@Test
void fanDeviceCapabilities() {
    var device = FanDevice.builder()
        .deviceId("f1").deviceClass(DeviceClass.FAN).label("Fan")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .on(true).speed(3).build();
    var caps = device.capabilities();
    assertThat(caps).containsEntry(DeviceEntity.CAP_AVAILABLE, true);
    assertThat(caps).containsEntry(FanDevice.CAP_ON, true);
    assertThat(caps).containsEntry(FanDevice.CAP_SPEED, 3);
    assertThat(caps).hasSize(3);
}
```

- [ ] **Step 2: Add tests to ToBuilderTest.java**

```java
@Test
void mediaPlayerDeviceToBuilderModifyVolume() {
    var original = MediaPlayerDevice.builder()
        .deviceId("mp1").deviceClass(DeviceClass.MEDIA_PLAYER).label("Player")
        .available(true).lastUpdated(NOW).tenancyId("t1")
        .playing(true).volume(80).build();
    MediaPlayerDevice modified = original.toBuilder().volume(60).build();
    assertThat(modified.volume()).hasValue(60);
    assertThat(modified.isPlaying()).isTrue();
    assertThat(modified).isInstanceOf(MediaPlayerDevice.class);
}

@Test
void fanDeviceToBuilderModifySpeed() {
    var original = FanDevice.builder()
        .deviceId("f1").deviceClass(DeviceClass.FAN).label("Fan")
        .available(true).lastUpdated(NOW).tenancyId("t1").on(true).speed(3).build();
    FanDevice modified = original.toBuilder().speed(5).build();
    assertThat(modified.speed()).hasValue(5);
    assertThat(modified.isOn()).isTrue();
    assertThat(modified).isInstanceOf(FanDevice.class);
}
```

- [ ] **Step 3: Run and confirm failures**

```bash
mvn --batch-mode test -pl iot-api -Dtest="CapabilitiesTest+ToBuilderTest"
```

Expected: MediaPlayerDevice and FanDevice tests FAIL.

- [ ] **Step 4: Add capabilities() and toBuilder() to MediaPlayerDevice.java**

Add import:
```java
import java.util.Map;
```

Inside MediaPlayerDevice class:

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_PLAYING, playing);
    caps.put(CAP_VOLUME, volume);
    return caps;
}

public MediaPlayerDevice.Builder toBuilder() {
    return MediaPlayerDevice.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .playing(playing).volume(volume);
}
```

- [ ] **Step 5: Add capabilities() and toBuilder() to FanDevice.java**

Add import:
```java
import java.util.Map;
```

Inside FanDevice class:

```java
@Override
public Map<String, Object> capabilities() {
    Map<String, Object> caps = super.capabilities();
    caps.put(CAP_ON, on);
    caps.put(CAP_SPEED, speed);
    return caps;
}

public FanDevice.Builder toBuilder() {
    return FanDevice.builder()
        .deviceId(deviceId()).deviceClass(deviceClass()).label(label())
        .available(available()).lastUpdated(lastUpdated()).tenancyId(tenancyId())
        .on(on).speed(speed);
}
```

- [ ] **Step 6: Run all iot-api tests**

```bash
mvn --batch-mode test -pl iot-api
```

Expected: All tests PASS (including all existing tests from C1).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/MediaPlayerDevice.java iot-api/src/main/java/io/casehub/iot/api/FanDevice.java iot-api/src/test/java/io/casehub/iot/api/CapabilitiesTest.java iot-api/src/test/java/io/casehub/iot/api/ToBuilderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): MediaPlayerDevice + FanDevice capabilities() + toBuilder() #2"
```

---

## Task 10: StateChangeEvent.deriveChangedCapabilities()

Static utility that diffs two DeviceEntity capability maps. Requires same concrete type — throws `IllegalArgumentException` otherwise. One-sided iteration is exhaustive because the same-type precondition guarantees identical key sets.

**Files:**
- Modify: `iot-api/src/main/java/io/casehub/iot/api/StateChangeEvent.java`
- Create: `iot-api/src/test/java/io/casehub/iot/api/DeriveChangedCapabilitiesTest.java`

- [ ] **Step 1: Create DeriveChangedCapabilitiesTest.java**

```java
package io.casehub.iot.api;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DeriveChangedCapabilitiesTest {

    static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    private SwitchDevice sw(boolean on) {
        return SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("S")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(on).build();
    }

    @Test
    void noChangeProducesEmptySet() {
        var device = sw(true);
        assertThat(StateChangeEvent.deriveChangedCapabilities(device, device)).isEmpty();
    }

    @Test
    void singleFieldChangeDectedCorrectly() {
        var before = sw(false);
        var after = sw(true);
        assertThat(StateChangeEvent.deriveChangedCapabilities(before, after))
            .containsExactly(SwitchDevice.CAP_ON);
    }

    @Test
    void availabilityChangeDetected() {
        var before = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("S")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
        var after = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("S")
            .available(false).lastUpdated(NOW).tenancyId("t1").on(false).build();
        assertThat(StateChangeEvent.deriveChangedCapabilities(before, after))
            .containsExactly(DeviceEntity.CAP_AVAILABLE);
    }

    @Test
    void nullToNonNullTransitionDetected() {
        var before = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        var after = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).brightness(200).build();
        assertThat(StateChangeEvent.deriveChangedCapabilities(before, after))
            .containsExactly(LightDevice.CAP_BRIGHTNESS);
    }

    @Test
    void nonNullToNullTransitionDetected() {
        var before = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).brightness(200).build();
        var after = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(true).build();
        assertThat(StateChangeEvent.deriveChangedCapabilities(before, after))
            .containsExactly(LightDevice.CAP_BRIGHTNESS);
    }

    @Test
    void multipleFieldChangesDetected() {
        var current = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
        var newCurrent = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
        var target = new Temperature(new BigDecimal("23"), Temperature.TemperatureUnit.CELSIUS);
        var before = ThermostatDevice.builder()
            .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("T")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .currentTemperature(current).targetTemperature(target).mode(ThermostatMode.HEAT).build();
        var after = before.toBuilder().currentTemperature(newCurrent).mode(ThermostatMode.COOL).build();
        var changed = StateChangeEvent.deriveChangedCapabilities(before, after);
        assertThat(changed).containsExactlyInAnyOrder(
            ThermostatDevice.CAP_CURRENT_TEMPERATURE, ThermostatDevice.CAP_MODE);
    }

    @Test
    void temperatureScaleInsensitiveComparison() {
        var t21 = new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS);
        var t21same = new Temperature(new BigDecimal("21.0"), Temperature.TemperatureUnit.CELSIUS);
        var target = new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS);
        var before = ThermostatDevice.builder()
            .deviceId("th1").deviceClass(DeviceClass.THERMOSTAT).label("T")
            .available(true).lastUpdated(NOW).tenancyId("t1")
            .currentTemperature(t21).targetTemperature(target).mode(ThermostatMode.HEAT).build();
        var after = before.toBuilder().currentTemperature(t21same).build();
        assertThat(StateChangeEvent.deriveChangedCapabilities(before, after)).isEmpty();
    }

    @Test
    void differentTypesThrowIllegalArgumentException() {
        var sw = SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("S")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
        var light = LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("L")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
        assertThatThrownBy(() -> StateChangeEvent.deriveChangedCapabilities(sw, light))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("SwitchDevice")
            .hasMessageContaining("LightDevice");
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-api -Dtest=DeriveChangedCapabilitiesTest
```

Expected: COMPILATION ERROR — `StateChangeEvent.deriveChangedCapabilities` does not exist.

- [ ] **Step 3: Add deriveChangedCapabilities() to StateChangeEvent.java**

Add imports to StateChangeEvent.java:
```java
import java.util.LinkedHashSet;
import java.util.Map;
```

Inside the `StateChangeEvent` record body, add:

```java
public static Set<String> deriveChangedCapabilities(
        DeviceEntity before, DeviceEntity after) {
    if (before.getClass() != after.getClass()) {
        throw new IllegalArgumentException(
            "Cannot derive changed capabilities across different types: "
            + before.getClass().getSimpleName() + " vs "
            + after.getClass().getSimpleName());
    }
    Map<String, Object> capsBefore = before.capabilities();
    Map<String, Object> capsAfter = after.capabilities();
    Set<String> changed = new LinkedHashSet<>();
    for (var entry : capsAfter.entrySet()) {
        Object prev = capsBefore.get(entry.getKey());
        if (!Objects.equals(prev, entry.getValue())) {
            changed.add(entry.getKey());
        }
    }
    return Set.copyOf(changed);
}
```

- [ ] **Step 4: Run all DeriveChangedCapabilitiesTest tests**

```bash
mvn --batch-mode test -pl iot-api -Dtest=DeriveChangedCapabilitiesTest
```

Expected: All 8 tests PASS.

- [ ] **Step 5: Run full iot-api test suite**

```bash
mvn --batch-mode test -pl iot-api
```

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/main/java/io/casehub/iot/api/StateChangeEvent.java iot-api/src/test/java/io/casehub/iot/api/DeriveChangedCapabilitiesTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-api): StateChangeEvent.deriveChangedCapabilities() #2"
```

---

## Task 11: CdiDeviceRegistryTest rewrite + delete DeviceRegistryContractTest

`CdiDeviceRegistryTest.stateChangeUpdatesRegistry()` currently calls the package-private `onStateChange()` method directly, bypassing CDI event infrastructure. Rewrite it to inject `Event<StateChangeEvent>` and use `fireAsync().join()`. Delete `DeviceRegistryContractTest` — its contract coverage is now provided by `MockDeviceRegistryTest` in iot-testing.

**Files:**
- Modify: `iot-api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java`
- Delete: `iot-api/src/test/java/io/casehub/iot/api/spi/DeviceRegistryContractTest.java`

- [ ] **Step 1: Delete DeviceRegistryContractTest.java**

```bash
rm /Users/mdproctor/claude/casehub/iot/iot-api/src/test/java/io/casehub/iot/api/spi/DeviceRegistryContractTest.java
```

- [ ] **Step 2: Rewrite stateChangeUpdatesRegistry() in CdiDeviceRegistryTest.java**

Replace the existing `stateChangeUpdatesRegistry()` test method. Also add the `Event<StateChangeEvent>` injection field. The full class after changes:

```java
package io.casehub.iot.spi;

import io.casehub.iot.api.*;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.api.spi.DeviceRegistry;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.inject.Alternative;
import jakarta.annotation.Priority;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CdiDeviceRegistryTest {

    private static final Instant NOW = Instant.parse("2026-06-07T10:00:00Z");

    @ApplicationScoped
    @Alternative
    @Priority(1)
    static class TestProvider implements DeviceProvider {
        @Override public String providerId() { return "test"; }
        @Override public List<DeviceEntity> discover() {
            return List.of(
                SwitchDevice.builder().deviceId("sw1").deviceClass(DeviceClass.SWITCH)
                    .label("Switch").available(true).lastUpdated(NOW).tenancyId("t1").on(true).build(),
                LightDevice.builder().deviceId("l1").deviceClass(DeviceClass.LIGHT)
                    .label("Light").available(true).lastUpdated(NOW).tenancyId("t2").on(true).brightness(200).build()
            );
        }
        @Override public CommandResult dispatch(DeviceCommand command) { return CommandResult.SENT; }
        @Override public ProviderStatus status() { return ProviderStatus.CONNECTED; }
    }

    @Inject
    DeviceRegistry registry;

    @Inject
    Event<StateChangeEvent> events;

    @Test
    void discoversDevicesAtStartup() {
        assertThat(registry.findAll()).hasSize(2);
        assertThat(registry.findById("sw1")).isPresent();
        assertThat(registry.findById("l1")).isPresent();
    }

    @Test
    void findByClassFiltersCorrectly() {
        assertThat(registry.findByClass(SwitchDevice.class)).hasSize(1);
        assertThat(registry.findByClass(LightDevice.class)).hasSize(1);
        assertThat(registry.findByClass(DeviceEntity.class)).hasSize(2);
    }

    @Test
    void findByTenancyIdFiltersCorrectly() {
        assertThat(registry.findByTenancyId("t1")).hasSize(1);
        assertThat(registry.findByTenancyId("t2")).hasSize(1);
        assertThat(registry.findByTenancyId("unknown")).isEmpty();
    }

    @Test
    void stateChangeUpdatesRegistry() throws Exception {
        var before = (SwitchDevice) registry.findById("sw1").orElseThrow();
        var after = before.toBuilder().on(false).lastUpdated(Instant.now()).build();

        events.fireAsync(new StateChangeEvent(
            before, after,
            StateChangeEvent.deriveChangedCapabilities(before, after),
            Instant.now(), "test"))
        .toCompletableFuture().join();

        var updated = (SwitchDevice) registry.findById("sw1").orElseThrow();
        assertThat(updated.isOn()).isFalse();
    }

    @Test
    void refreshRebuildsDeviceMap() {
        registry.refresh();
        assertThat(registry.findAll()).hasSize(2);
    }
}
```

- [ ] **Step 3: Run CdiDeviceRegistryTest**

```bash
mvn --batch-mode test -pl iot-api -Dtest=CdiDeviceRegistryTest
```

Expected: All 5 tests PASS.

- [ ] **Step 4: Run full iot-api test suite**

```bash
mvn --batch-mode test -pl iot-api
```

Expected: All tests PASS. DeviceRegistryContractTest no longer present.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-api/src/test/java/io/casehub/iot/spi/CdiDeviceRegistryTest.java
git -C /Users/mdproctor/claude/casehub/iot rm iot-api/src/test/java/io/casehub/iot/api/spi/DeviceRegistryContractTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "refactor(iot-api): rewrite stateChangeUpdatesRegistry() via CDI Event; delete DeviceRegistryContractTest #2"
```

---

## Task 12: iot-testing pom.xml — dependencies + Jandex plugin

iot-testing needs three additions: `quarkus-junit` (test) for `@QuarkusTest`, `assertj-core` (test) for assertions, and the Jandex Maven plugin to generate a CDI index so Quarkus discovers `StateChangeEventPublisher` as an `@ApplicationScoped` bean.

**Files:**
- Modify: `iot-testing/pom.xml`

- [ ] **Step 1: Update iot-testing/pom.xml**

Replace the entire content of `iot-testing/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-iot-parent</artifactId>
        <version>0.1-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-iot-testing</artifactId>
    <name>CaseHub IoT — Testing</name>
    <description>MockDeviceProvider, fixture devices, StateChangeEventPublisher — test scope only</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-iot-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

- [ ] **Step 2: Verify the parent BOM manages jandex-maven-plugin version**

```bash
mvn --batch-mode help:effective-pom -pl iot-testing | grep -A5 "jandex"
```

If the version is not managed by the parent BOM, add `<version>3.1.8</version>` (or the latest stable) inside the `<plugin>` block. Check https://mvnrepository.com/artifact/io.smallrye/jandex-maven-plugin for the current version if needed.

- [ ] **Step 3: Verify iot-testing compiles**

```bash
mvn --batch-mode compile -pl iot-testing
```

Expected: BUILD SUCCESS (no sources yet, that's fine).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-testing/pom.xml
git -C /Users/mdproctor/claude/casehub/iot commit -m "build(iot-testing): add quarkus-junit, assertj, Jandex plugin #2"
```

---

## Task 13: MockDeviceProvider

Plain Java POJO. No CDI dependency. Records dispatched commands, controls status and dispatch result.

**Files:**
- Create: `iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceProvider.java`
- Create: `iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceProviderTest.java`

- [ ] **Step 1: Create MockDeviceProviderTest.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class MockDeviceProviderTest {

    static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");

    MockDeviceProvider provider;

    @BeforeEach
    void setUp() {
        provider = new MockDeviceProvider("test");
    }

    private SwitchDevice sw(String id) {
        return SwitchDevice.builder()
            .deviceId(id).deviceClass(DeviceClass.SWITCH).label("S")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build();
    }

    @Test
    void providerIdReturned() {
        assertThat(provider.providerId()).isEqualTo("test");
    }

    @Test
    void discoverReturnsAddedDevices() {
        provider.addDevice(sw("sw1"));
        assertThat(provider.discover()).hasSize(1);
        assertThat(provider.discover().get(0).deviceId()).isEqualTo("sw1");
    }

    @Test
    void discoverReturnsEmptyWhenNothingAdded() {
        assertThat(provider.discover()).isEmpty();
    }

    @Test
    void addDeviceOverwritesExistingByDeviceId() {
        provider.addDevice(sw("sw1"));
        var updated = sw("sw1").toBuilder().available(false).build();
        provider.addDevice(updated);
        assertThat(provider.discover()).hasSize(1);
        assertThat(provider.discover().get(0).available()).isFalse();
    }

    @Test
    void removeDeviceRemovesFromDiscovery() {
        provider.addDevice(sw("sw1"));
        provider.removeDevice("sw1");
        assertThat(provider.discover()).isEmpty();
    }

    @Test
    void clearRemovesAllDevices() {
        provider.addDevice(sw("sw1"));
        provider.addDevice(sw("sw2"));
        provider.clear();
        assertThat(provider.discover()).isEmpty();
    }

    @Test
    void dispatchRecordsCommand() {
        var cmd = DeviceCommand.turnOff("sw1", "actor", "corr");
        provider.dispatch(cmd);
        assertThat(provider.dispatchedCommands()).containsExactly(cmd);
    }

    @Test
    void dispatchDefaultResultIsSent() {
        assertThat(provider.dispatch(DeviceCommand.turnOff("sw1", "a", "c")))
            .isEqualTo(CommandResult.SENT);
    }

    @Test
    void dispatchReturnsConfiguredResult() {
        provider.setDispatchResult(CommandResult.FAILED);
        assertThat(provider.dispatch(DeviceCommand.turnOff("sw1", "a", "c")))
            .isEqualTo(CommandResult.FAILED);
    }

    @Test
    void statusDefaultIsConnected() {
        assertThat(provider.status()).isEqualTo(ProviderStatus.CONNECTED);
    }

    @Test
    void statusReturnsConfiguredStatus() {
        provider.setStatus(ProviderStatus.DISCONNECTED);
        assertThat(provider.status()).isEqualTo(ProviderStatus.DISCONNECTED);
    }

    @Test
    void clearDispatchedCommandsClearsLog() {
        provider.dispatch(DeviceCommand.turnOff("sw1", "a", "c"));
        provider.clearDispatchedCommands();
        assertThat(provider.dispatchedCommands()).isEmpty();
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=MockDeviceProviderTest
```

Expected: COMPILATION ERROR — `MockDeviceProvider` does not exist.

- [ ] **Step 3: Create MockDeviceProvider.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.CommandResult;
import io.casehub.iot.api.DeviceCommand;
import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.ProviderStatus;
import io.casehub.iot.api.spi.DeviceProvider;
import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class MockDeviceProvider implements DeviceProvider {

    private final String providerId;
    private final Map<String, DeviceEntity> devices = new LinkedHashMap<>();
    private final List<DeviceCommand> dispatchedCommands = new ArrayList<>();
    private ProviderStatus status = ProviderStatus.CONNECTED;
    private CommandResult dispatchResult = CommandResult.SENT;

    public MockDeviceProvider(String providerId) {
        this.providerId = providerId;
    }

    @Override
    public String providerId() { return providerId; }

    @Override
    public List<DeviceEntity> discover() {
        return List.copyOf(devices.values());
    }

    @Override
    public CommandResult dispatch(DeviceCommand command) {
        dispatchedCommands.add(command);
        return dispatchResult;
    }

    @Override
    public ProviderStatus status() { return status; }

    public void addDevice(DeviceEntity device) {
        devices.put(device.deviceId(), device);
    }

    public void removeDevice(String deviceId) {
        devices.remove(deviceId);
    }

    public void clear() {
        devices.clear();
    }

    public void setStatus(ProviderStatus status) {
        this.status = status;
    }

    public void setDispatchResult(CommandResult dispatchResult) {
        this.dispatchResult = dispatchResult;
    }

    public List<DeviceCommand> dispatchedCommands() {
        return Collections.unmodifiableList(dispatchedCommands);
    }

    public void clearDispatchedCommands() {
        dispatchedCommands.clear();
    }
}
```

- [ ] **Step 4: Run tests**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=MockDeviceProviderTest
```

Expected: All 12 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceProvider.java iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceProviderTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-testing): MockDeviceProvider #2"
```

---

## Task 14: MockDeviceRegistry

Plain Java, no CDI. Simple in-memory registry for unit tests that don't need a CDI container. `refresh()` is a no-op — populated programmatically via `addDevice()`.

**Files:**
- Create: `iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceRegistry.java`
- Create: `iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceRegistryTest.java`

- [ ] **Step 1: Create MockDeviceRegistryTest.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class MockDeviceRegistryTest {

    static final Instant NOW = Instant.parse("2026-01-01T00:00:00Z");

    MockDeviceRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new MockDeviceRegistry();
        registry.addDevice(SwitchDevice.builder()
            .deviceId("sw1").deviceClass(DeviceClass.SWITCH).label("Switch")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build());
        registry.addDevice(LightDevice.builder()
            .deviceId("l1").deviceClass(DeviceClass.LIGHT).label("Light")
            .available(true).lastUpdated(NOW).tenancyId("t1").on(false).build());
    }

    @Test
    void findByIdReturnsKnownDevice() {
        assertThat(registry.findById("sw1")).isPresent();
        assertThat(registry.findById("sw1").get().deviceId()).isEqualTo("sw1");
    }

    @Test
    void findByIdReturnsEmptyForUnknown() {
        assertThat(registry.findById("unknown")).isEmpty();
    }

    @Test
    void findByClassFiltersToConcreteType() {
        assertThat(registry.findByClass(SwitchDevice.class)).hasSize(1);
        assertThat(registry.findByClass(LightDevice.class)).hasSize(1);
        assertThat(registry.findByClass(ThermostatDevice.class)).isEmpty();
        assertThat(registry.findByClass(DeviceEntity.class)).hasSize(2);
    }

    @Test
    void findByTenancyIdFilters() {
        assertThat(registry.findByTenancyId("t1")).hasSize(2);
        assertThat(registry.findByTenancyId("other")).isEmpty();
    }

    @Test
    void findAllReturnsAllDevices() {
        assertThat(registry.findAll()).hasSize(2);
    }

    @Test
    void refreshIsNoOp() {
        registry.refresh();
        assertThat(registry.findAll()).hasSize(2);
    }

    @Test
    void clearRemovesAllDevices() {
        registry.clear();
        assertThat(registry.findAll()).isEmpty();
    }

    @Test
    void addDevicesVarargs() {
        registry.clear();
        registry.addDevices(
            SwitchDevice.builder().deviceId("sw1").deviceClass(DeviceClass.SWITCH)
                .label("S").available(true).lastUpdated(NOW).tenancyId("t1").on(false).build(),
            LightDevice.builder().deviceId("l1").deviceClass(DeviceClass.LIGHT)
                .label("L").available(true).lastUpdated(NOW).tenancyId("t1").on(false).build());
        assertThat(registry.findAll()).hasSize(2);
    }

    @Test
    void addDevicesList() {
        registry.clear();
        registry.addDevices(Fixtures.standardHome());
        assertThat(registry.findAll()).hasSize(10);
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=MockDeviceRegistryTest
```

Expected: COMPILATION ERROR — `MockDeviceRegistry` and `Fixtures` do not exist.

- [ ] **Step 3: Create MockDeviceRegistry.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.spi.DeviceRegistry;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

public class MockDeviceRegistry implements DeviceRegistry {

    private final Map<String, DeviceEntity> devices = new LinkedHashMap<>();

    @Override
    public Optional<DeviceEntity> findById(String deviceId) {
        return Optional.ofNullable(devices.get(deviceId));
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends DeviceEntity> List<T> findByClass(Class<T> deviceClass) {
        return devices.values().stream()
            .filter(deviceClass::isInstance)
            .map(d -> (T) d)
            .toList();
    }

    @Override
    public List<DeviceEntity> findByTenancyId(String tenancyId) {
        return devices.values().stream()
            .filter(d -> d.tenancyId().equals(tenancyId))
            .toList();
    }

    @Override
    public List<DeviceEntity> findAll() {
        return List.copyOf(devices.values());
    }

    @Override
    public void refresh() { /* no-op — populated programmatically */ }

    public void addDevice(DeviceEntity device) {
        devices.put(device.deviceId(), device);
    }

    public void addDevices(DeviceEntity... devs) {
        Arrays.stream(devs).forEach(this::addDevice);
    }

    public void addDevices(List<DeviceEntity> devs) {
        devs.forEach(this::addDevice);
    }

    public void clear() {
        devices.clear();
    }
}
```

Note: `addDevicesList` test requires `Fixtures` (Task 15). Leave it commented for now if needed, or create a stub Fixtures first. If building in order, complete Task 15 before verifying the `addDevicesList` test.

- [ ] **Step 4: Commit MockDeviceRegistry (before Fixtures exist)**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-testing/src/main/java/io/casehub/iot/testing/MockDeviceRegistry.java iot-testing/src/test/java/io/casehub/iot/testing/MockDeviceRegistryTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-testing): MockDeviceRegistry #2"
```

---

## Task 15: Fixtures

Static factory methods for a standard home device set. All 10 device classes covered. Fixed IDs, fixed tenant, deterministic EPOCH timestamp.

**Files:**
- Create: `iot-testing/src/main/java/io/casehub/iot/testing/Fixtures.java`
- Create: `iot-testing/src/test/java/io/casehub/iot/testing/FixturesTest.java`

- [ ] **Step 1: Create FixturesTest.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.*;
import org.junit.jupiter.api.Test;
import java.util.stream.Collectors;
import static org.assertj.core.api.Assertions.assertThat;

class FixturesTest {

    @Test
    void allTenFactoriesProduceDevicesWithCorrectIds() {
        assertThat(Fixtures.hallwaySwitch().deviceId()).isEqualTo("switch-hallway-1");
        assertThat(Fixtures.livingRoomLight().deviceId()).isEqualTo("light-living-1");
        assertThat(Fixtures.livingRoomThermostat().deviceId()).isEqualTo("thermostat-living-1");
        assertThat(Fixtures.outdoorTemperature().deviceId()).isEqualTo("sensor-outdoor-1");
        assertThat(Fixtures.frontDoorPresence().deviceId()).isEqualTo("presence-front-1");
        assertThat(Fixtures.solarPanel().deviceId()).isEqualTo("power-solar-1");
        assertThat(Fixtures.frontDoorLock().deviceId()).isEqualTo("lock-front-1");
        assertThat(Fixtures.bedroomBlinds().deviceId()).isEqualTo("cover-bedroom-1");
        assertThat(Fixtures.livingRoomSpeaker().deviceId()).isEqualTo("media-living-1");
        assertThat(Fixtures.bedroomFan().deviceId()).isEqualTo("fan-bedroom-1");
    }

    @Test
    void allDevicesUseDefaultTenantAndEpoch() {
        for (var device : Fixtures.standardHome()) {
            assertThat(device.tenancyId()).isEqualTo(Fixtures.DEFAULT_TENANT);
            assertThat(device.lastUpdated()).isEqualTo(Fixtures.EPOCH);
            assertThat(device.available()).isTrue();
        }
    }

    @Test
    void factoriesReturnFreshInstances() {
        assertThat(Fixtures.hallwaySwitch()).isNotSameAs(Fixtures.hallwaySwitch());
        assertThat(Fixtures.frontDoorLock()).isNotSameAs(Fixtures.frontDoorLock());
    }

    @Test
    void standardHomeContainsTenDistinctDevices() {
        var home = Fixtures.standardHome();
        assertThat(home).hasSize(10);
        var ids = home.stream().map(DeviceEntity::deviceId).collect(Collectors.toSet());
        assertThat(ids).hasSize(10);
    }

    @Test
    void standardHomeCoversAllDeviceClasses() {
        var classes = Fixtures.standardHome().stream()
            .map(DeviceEntity::deviceClass)
            .collect(Collectors.toSet());
        assertThat(classes).containsExactlyInAnyOrder(DeviceClass.values());
    }

    @Test
    void initialDomainStatesAreCorrect() {
        assertThat(Fixtures.hallwaySwitch().isOn()).isFalse();
        assertThat(Fixtures.livingRoomLight().isOn()).isFalse();
        assertThat(Fixtures.livingRoomLight().brightness()).isEmpty();
        assertThat(Fixtures.livingRoomThermostat().mode()).isEqualTo(ThermostatMode.HEAT);
        assertThat(Fixtures.frontDoorPresence().isPresent()).isFalse();
        assertThat(Fixtures.frontDoorPresence().lastSeen()).isEqualTo(Fixtures.EPOCH);
        assertThat(Fixtures.frontDoorLock().isLocked()).isTrue();
        assertThat(Fixtures.bedroomBlinds().position()).isEqualTo(0);
        assertThat(Fixtures.bedroomBlinds().isMoving()).isFalse();
        assertThat(Fixtures.livingRoomSpeaker().isPlaying()).isFalse();
        assertThat(Fixtures.bedroomFan().isOn()).isFalse();
    }

    @Test
    void toBuilderWorksWithFixtures() {
        var original = Fixtures.frontDoorLock();
        LockDevice unlocked = original.toBuilder().locked(false).build();
        assertThat(unlocked.isLocked()).isFalse();
        assertThat(unlocked.deviceId()).isEqualTo("lock-front-1");
    }

    @Test
    void availabilityVariantViaToBuilder() {
        var offline = Fixtures.hallwaySwitch().toBuilder().available(false).build();
        assertThat(offline.available()).isFalse();
        assertThat(offline.deviceId()).isEqualTo("switch-hallway-1");
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=FixturesTest
```

Expected: COMPILATION ERROR — `Fixtures` does not exist.

- [ ] **Step 3: Create Fixtures.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

public final class Fixtures {

    public static final String DEFAULT_TENANT = "default-tenant";
    public static final Instant EPOCH = Instant.parse("2026-01-01T00:00:00Z");

    private Fixtures() {}

    public static SwitchDevice hallwaySwitch() {
        return SwitchDevice.builder()
            .deviceId("switch-hallway-1").deviceClass(DeviceClass.SWITCH)
            .label("Hallway Switch").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).on(false).build();
    }

    public static LightDevice livingRoomLight() {
        return LightDevice.builder()
            .deviceId("light-living-1").deviceClass(DeviceClass.LIGHT)
            .label("Living Room Light").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).on(false).build();
    }

    public static ThermostatDevice livingRoomThermostat() {
        return ThermostatDevice.builder()
            .deviceId("thermostat-living-1").deviceClass(DeviceClass.THERMOSTAT)
            .label("Living Room Thermostat").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT)
            .currentTemperature(new Temperature(new BigDecimal("21"), Temperature.TemperatureUnit.CELSIUS))
            .targetTemperature(new Temperature(new BigDecimal("22"), Temperature.TemperatureUnit.CELSIUS))
            .mode(ThermostatMode.HEAT).build();
    }

    public static SensorDevice outdoorTemperature() {
        return SensorDevice.builder()
            .deviceId("sensor-outdoor-1").deviceClass(DeviceClass.SENSOR)
            .label("Outdoor Temperature").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT)
            .sensorType(SensorType.TEMPERATURE)
            .numericValue(new BigDecimal("15")).unit("C").build();
    }

    public static PresenceSensor frontDoorPresence() {
        return PresenceSensor.builder()
            .deviceId("presence-front-1").deviceClass(DeviceClass.PRESENCE_SENSOR)
            .label("Front Door Presence").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).present(false).lastSeen(EPOCH).build();
    }

    public static PowerSensor solarPanel() {
        return PowerSensor.builder()
            .deviceId("power-solar-1").deviceClass(DeviceClass.POWER_SENSOR)
            .label("Solar Panel").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT)
            .power(BigDecimal.ZERO).energy(BigDecimal.ZERO).build();
    }

    public static LockDevice frontDoorLock() {
        return LockDevice.builder()
            .deviceId("lock-front-1").deviceClass(DeviceClass.LOCK)
            .label("Front Door Lock").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).locked(true).build();
    }

    public static CoverDevice bedroomBlinds() {
        return CoverDevice.builder()
            .deviceId("cover-bedroom-1").deviceClass(DeviceClass.COVER)
            .label("Bedroom Blinds").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).position(0).moving(false).build();
    }

    public static MediaPlayerDevice livingRoomSpeaker() {
        return MediaPlayerDevice.builder()
            .deviceId("media-living-1").deviceClass(DeviceClass.MEDIA_PLAYER)
            .label("Living Room Speaker").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).playing(false).build();
    }

    public static FanDevice bedroomFan() {
        return FanDevice.builder()
            .deviceId("fan-bedroom-1").deviceClass(DeviceClass.FAN)
            .label("Bedroom Fan").available(true).lastUpdated(EPOCH)
            .tenancyId(DEFAULT_TENANT).on(false).build();
    }

    public static List<DeviceEntity> standardHome() {
        return List.of(
            hallwaySwitch(), livingRoomLight(), livingRoomThermostat(),
            outdoorTemperature(), frontDoorPresence(), solarPanel(),
            frontDoorLock(), bedroomBlinds(), livingRoomSpeaker(), bedroomFan());
    }
}
```

- [ ] **Step 4: Run all iot-testing unit tests**

```bash
mvn --batch-mode test -pl iot-testing -Dtest="MockDeviceProviderTest+MockDeviceRegistryTest+FixturesTest"
```

Expected: All tests PASS (including MockDeviceRegistryTest.addDevicesList which now compiles against Fixtures).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-testing/src/main/java/io/casehub/iot/testing/Fixtures.java iot-testing/src/test/java/io/casehub/iot/testing/FixturesTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-testing): Fixtures — 10 device factory methods + standardHome() #2"
```

---

## Task 16: StateChangeEventPublisher + @QuarkusTest

CDI `@ApplicationScoped` bean that fires `StateChangeEvent` via `Event.fireAsync()`. Tests use `@QuarkusTest`. The `CapturedEvents` inner observer class records all events for assertion. **Both `CapturedEvents` and `StateChangeEventPublisher` must be CDI-discovered — the Jandex index from Task 12 enables this.**

**Files:**
- Create: `iot-testing/src/main/java/io/casehub/iot/testing/StateChangeEventPublisher.java`
- Create: `iot-testing/src/test/java/io/casehub/iot/testing/StateChangeEventPublisherTest.java`

- [ ] **Step 1: Create StateChangeEventPublisherTest.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.*;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class StateChangeEventPublisherTest {

    @ApplicationScoped
    static class CapturedEvents {
        private final List<StateChangeEvent> received = new CopyOnWriteArrayList<>();

        void onEvent(@ObservesAsync StateChangeEvent event) {
            received.add(event);
        }

        List<StateChangeEvent> getAll() { return List.copyOf(received); }
        void clear() { received.clear(); }
    }

    @Inject StateChangeEventPublisher publisher;
    @Inject CapturedEvents captured;

    @BeforeEach
    void resetObserver() {
        captured.clear();
    }

    @Test
    void publishFiresEventWithAutoCalculatedChangedCapabilities() throws Exception {
        var before = Fixtures.hallwaySwitch();
        var after = before.toBuilder().on(true).build();

        publisher.publish(before, after, "test").toCompletableFuture().join();

        assertThat(captured.getAll()).hasSize(1);
        var event = captured.getAll().get(0);
        assertThat(event.changedCapabilities()).containsExactly(SwitchDevice.CAP_ON);
        assertThat(event.providerId()).isEqualTo("test");
        assertThat(event.before()).isEqualTo(before);
        assertThat(event.after()).isEqualTo(after);
    }

    @Test
    void publishWithNoChangeProducesEmptyChangedCapabilities() throws Exception {
        var device = Fixtures.frontDoorLock();

        publisher.publish(device, device.toBuilder().build(), "test")
            .toCompletableFuture().join();

        assertThat(captured.getAll().get(0).changedCapabilities()).isEmpty();
    }

    @Test
    void publishWithAvailabilityChangeIncludesCapAvailable() throws Exception {
        var before = Fixtures.hallwaySwitch();
        var after = before.toBuilder().available(false).build();

        publisher.publish(before, after, "test").toCompletableFuture().join();

        assertThat(captured.getAll().get(0).changedCapabilities())
            .containsExactly(DeviceEntity.CAP_AVAILABLE);
    }

    @Test
    void multiplePublishesAreAllCaptured() throws Exception {
        var sw = Fixtures.hallwaySwitch();
        publisher.publish(sw, sw.toBuilder().on(true).build(), "test")
            .toCompletableFuture().join();
        publisher.publish(sw, sw.toBuilder().available(false).build(), "test")
            .toCompletableFuture().join();

        assertThat(captured.getAll()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run and confirm compilation failure**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=StateChangeEventPublisherTest
```

Expected: COMPILATION ERROR — `StateChangeEventPublisher` does not exist.

- [ ] **Step 3: Create StateChangeEventPublisher.java**

```java
package io.casehub.iot.testing;

import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.StateChangeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.concurrent.CompletionStage;

@ApplicationScoped
public class StateChangeEventPublisher {

    @Inject
    Event<StateChangeEvent> event;

    public CompletionStage<StateChangeEvent> publish(
            DeviceEntity before, DeviceEntity after, String providerId) {
        var sce = new StateChangeEvent(
            before, after,
            StateChangeEvent.deriveChangedCapabilities(before, after),
            Instant.now(), providerId);
        return event.fireAsync(sce);
    }
}
```

- [ ] **Step 4: Run StateChangeEventPublisherTest**

```bash
mvn --batch-mode test -pl iot-testing -Dtest=StateChangeEventPublisherTest
```

Expected: All 4 tests PASS.

If beans are not discovered (CDI injection fails): verify the Jandex index was generated at `iot-testing/target/classes/META-INF/jandex.idx`. If missing, re-run `mvn compile -pl iot-testing` first.

- [ ] **Step 5: Run complete iot-testing test suite**

```bash
mvn --batch-mode test -pl iot-testing
```

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/iot add iot-testing/src/main/java/io/casehub/iot/testing/StateChangeEventPublisher.java iot-testing/src/test/java/io/casehub/iot/testing/StateChangeEventPublisherTest.java
git -C /Users/mdproctor/claude/casehub/iot commit -m "feat(iot-testing): StateChangeEventPublisher — CDI async event firing #2"
```

---

## Task 17: Full build and push

Verify both modules build cleanly together and push the branch.

**Files:** None (verification only)

- [ ] **Step 1: Full install from root**

```bash
mvn --batch-mode install -pl iot-api,iot-testing
```

Expected: BUILD SUCCESS. Both modules compile, all tests pass.

- [ ] **Step 2: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/iot push -u origin issue-2-test-infrastructure
```

Expected: Branch pushed to origin.

- [ ] **Step 3: Push workspace branch**

```bash
git -C /Users/mdproctor/claude/public/casehub-iot push
```

Expected: Branch pushed to origin.

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Temperature.equals() scale-insensitive fix | Task 1 |
| DeviceEntity.CAP_AVAILABLE + concrete capabilities() | Task 2 |
| All 10 subclasses override capabilities() | Tasks 3–9 |
| capabilities() always allocates fresh map | Task 2 (test: capabilitiesAllocatesFreshMapEachCall) |
| toBuilder() on all 10 subclasses (not on DeviceEntity) | Tasks 3–9 |
| StateChangeEvent.deriveChangedCapabilities() + type check | Task 10 |
| StateChangeEvent.of() NOT added (constructor used directly) | Tasks 10, 16 ✓ |
| CdiDeviceRegistryTest rewrite via Event<StateChangeEvent> + join | Task 11 |
| DeviceRegistryContractTest deleted | Task 11 |
| iot-testing pom.xml: quarkus-junit, assertj, Jandex | Task 12 |
| MockDeviceProvider — full API | Task 13 |
| MockDeviceRegistry — full API | Task 14 |
| Fixtures — 10 factories + standardHome() + correct initial states | Task 15 |
| StateChangeEventPublisher @ApplicationScoped + CompletionStage | Task 16 |
| @BeforeEach state bleed note | Task 16 (CapturedEvents.clear() in @BeforeEach) |
| ARC42STORIES update noted | Consequential — done at work-end |

**Placeholder scan:** No TBD, no TODO, no "similar to Task N", no vague steps. All code blocks are complete.

**Type consistency:** `StateChangeEvent.deriveChangedCapabilities()` used consistently in Tasks 10, 11, 16. `toBuilder()` returns concrete typed builder in all 10 device type tasks, not DeviceEntity.Builder. `Fixtures.standardHome()` returns `List<DeviceEntity>` — consistent with MockDeviceRegistry.addDevices(List). `CompletionStage<StateChangeEvent>` return type on publisher — used correctly in test `.toCompletableFuture().join()`.
