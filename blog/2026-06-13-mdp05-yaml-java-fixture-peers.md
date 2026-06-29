---
layout: post
title: "YAML and Java as Fixture Peers"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [testing, yaml, spi, fixtures, serviceloader]
---

C2 shipped Java fixture factories — `Fixtures.hallwaySwitch()`, `Fixtures.standardHome()`, ten device factory methods covering every `DeviceClass`. YAML loading was deferred to #8 as additive work. The decision was right about ordering — Java factories needed to ship first so the builder API, `toBuilder()`, and `capabilities()` could be validated against real test usage. But the deferral was about sequencing, not about YAML being second-class.

The CaseHub platform treats YAML and Java DSL as paired authoring paths. The `case-definition-layers` protocol in casehub-engine establishes this: YAML resource files and fluent Java DSL builders are equal production-grade paths to `CaseDefinition`. Both produce the same canonical model. The IoT fixture loader follows the same principle — `DeviceFixtureLoader.load("fixtures/standard-home.yaml")` and `Fixtures.standardHome()` produce identical `List<DeviceEntity>`.

## The two-layer mapper

The design mirrors engine's `CaseDefinitionYamlMapper` — YAML text → intermediate representation → canonical model via builders. For IoT fixtures the intermediate layer is Jackson's `JsonNode` tree rather than generated schema classes. The device hierarchy is flat enough that custom DTOs would be unnecessary complexity.

A `DeviceTypeHandler` SPI takes a `JsonNode` and a `DeviceFixtureDefaults` and returns a `DeviceEntity`. One handler per concrete device type. A static interface method — `applyCommonFields` — populates the six base `DeviceEntity` fields from the node, so each handler only reads its type-specific fields. The type discriminator in YAML is lowercase `DeviceClass` for common types (`switch`, `thermostat`, `presence_sensor`) and `provider:base_class` for supplement types (`openhab:light`, `homeassistant:thermostat`).

## The `protected` Builder surprise

`DeviceEntity.Builder` was `protected abstract static class`. Standard for builder hierarchies — consumers construct via concrete subclass builders, never the abstract base. But `applyCommonFields` uses `<B extends DeviceEntity.Builder<?, B>>` as a generic bound, and Java's access rules apply to type references in generic bounds, not just method calls. A static interface method in `io.casehub.iot.testing` can't reference a protected class in `io.casehub.iot.api` — even though every concrete builder it will actually receive is public.

The error message — "Builder has protected access in DeviceEntity" — reads like a field access problem. It isn't. The fix is making `Builder` public. The builder's setter methods were already public; the nested class visibility was the only constraint, and it was wrong. This is a public API module — the builder abstraction is part of the API surface whether the access modifier says so or not.

## ServiceLoader for supplement types

The testing module can't depend on the provider modules. HomeAssistant and OpenHAB supplement types (`HomeAssistantLight`, `OpenHabThermostat`, etc.) need their own YAML handlers, but those handlers live in the provider JARs. ServiceLoader bridges this: each provider module ships `DeviceTypeHandler` implementations registered via `META-INF/services`, discovered automatically when the provider is on the test classpath.

The dependency from provider modules to `casehub-iot-testing` uses `<optional>true</optional>` — provider modules compile against the `DeviceTypeHandler` interface, handler classes ship in the provider JAR, but downstream consumers never get `iot-testing` transitively. At test time both the interface and implementations are on the classpath. At runtime the handler classes are inert dead code.

All 16 handlers — 10 common, 3 HomeAssistant, 3 OpenHAB — register through the same ServiceLoader mechanism. No distinction between "built-in" and "supplement" in how they're discovered.

## Equivalence as the guarantee

The structural guarantee is an equivalence test: load `standard-home.yaml`, call `Fixtures.standardHome()`, compare with AssertJ's recursive comparison. If either authoring path produces a different device set, the test fails. Each provider module ships its own equivalence test for supplement types.

One subtlety with the comparison: `Temperature.equals()` uses `BigDecimal.compareTo()` for scale-insensitive equality — a deliberate C2 fix. But AssertJ's `usingRecursiveComparison()` uses `BigDecimal.equals()`, which is scale-sensitive. `new BigDecimal("21")` and `new BigDecimal("21.0")` don't match under recursive comparison even though they represent the same temperature. Jackson YAML parses `value: 21.0` as scale 1; the Java factory uses `new BigDecimal("21")` at scale 0. The fix is `withComparatorForType(BigDecimal::compareTo, BigDecimal.class)` — aligning the test machinery with the platform's own equality semantics.

YAML fixture loading is the kind of feature that makes you wonder why it wasn't first. The answer is that it couldn't be — the builder API, the value types, the supplement pattern all needed to exist and be validated before a serialization path could target them. Now both authoring paths exist, and the equivalence tests prove they stay in sync.