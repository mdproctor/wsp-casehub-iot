---
layout: post
title: "What the test harness taught us about the type system"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [java, quarkus, iot, testing, design]
---

Chapter 2 was supposed to be the easy one. Chapter 1 built the type system and SPI contracts; Chapter 2 just needed a mock provider, some fixtures, and a CDI helper to fire events. The design review had other ideas.

The first thing I wanted out of the test infrastructure was automatic derivation of `changedCapabilities`. In the real providers — Home Assistant and OpenHAB coming in C3 and C4 — each platform has its own way of telling you what changed, and we'd compute the diff from that. In tests, you'd do it by hand. That seemed wrong. So the design question became: how does an event publisher know which fields changed between two device snapshots?

The answer I landed on was to make each `DeviceEntity` subclass self-describing. Each one implements `capabilities()`, returning a map of its capability field names to current values. The publisher diffs the two maps. Straightforward enough — except for one wrinkle I didn't see coming.

`Temperature` is a Java record. Records generate `equals()` automatically, and the auto-generated version delegates to `BigDecimal.equals()`. `BigDecimal.equals()` is scale-sensitive: `new BigDecimal("21")` and `new BigDecimal("21.0")` are not equal, even though they're the same temperature. If a provider receives the same reading with different decimal precision — which is entirely plausible given how IoT platforms serialize floats — the capability diff would flag "currentTemperature changed" for a thermostat that didn't change at all.

What made this hard to catch: the existing `TemperatureTest` only tested equality using `BigDecimal.valueOf(20)` for both sides. `BigDecimal.valueOf()` always produces scale 0, so both instances had identical scale. The test passed. The bug was invisible until we looked at how `Temperature` values would appear in capability maps compared with `Objects.equals()`. The fix was straightforward once found — override `equals()` to use `compareTo()` and `hashCode()` to use `stripTrailingZeros()` — but it's the kind of thing you'd only catch if you thought about where equality was actually going to be exercised.

The `capabilities()` design also surfaced a question I hadn't thought to ask: what happens when a device goes offline? Availability — whether a device is reachable at all — is a runtime state that changes. A smart plug losing power should generate a `StateChangeEvent` with `"available"` in `changedCapabilities`. Without `CAP_AVAILABLE` in the base class, that event would arrive with an empty capability set. Anyone pattern-matching on `changedCapabilities` to decide what to do would miss it entirely. So `available` went into the base class `capabilities()` return, and every subclass picks it up via `super.capabilities()`.

The mutable return from `capabilities()` is the one part of this design that feels wrong until you understand what it's for. The method returns a fresh `LinkedHashMap` each time — not a copy, a genuinely mutable map. That's intentional. In C3 and C4, when we add vendor supplement types like `HomeAssistantLight extends LightDevice`, those supplements need to add their own capability fields. They call `super.capabilities()` and add their entries to the returned map. If the base returned an immutable copy, the chain would break. The mutable return is a protocol, not an oversight.

`toBuilder()` emerged from the same place. State transitions are the fundamental operation in IoT: "the same device, but with this field changed." Without `toBuilder()`, you rebuild from scratch every time, copying six base fields by hand. I'd been assuming that was fine — the test infrastructure would handle it — but the design review made clear this belongs on the concrete classes. `LightDevice.toBuilder()` returns `LightDevice.Builder`. If we'd put it on the abstract base, we'd get `Builder<?, ?>` back, and `build()` would return `DeviceEntity`. The type information would evaporate exactly when you needed it.

One decision from the ARC42STORIES didn't survive contact with reality: YAML fixtures. The original plan called for `home-standard-fixtures.yaml`, on the theory that non-developers could inspect and modify it. There are no non-developers in the audience for `casehub-iot-testing`. The consumers are engineers writing `@QuarkusTest` code against a Quarkus CDI container. Java factories — ten methods on a `Fixtures` class, one per device class — give you type safety, IDE navigation, and the same `toBuilder()` integration the rest of the code uses. YAML would have needed a parser dependency and a schema, and it would have fallen over the moment vendor supplement types arrived in C3.

The other thing worth noting: the Jandex index. `StateChangeEventPublisher` is an `@ApplicationScoped` CDI bean in `iot-testing`. For Quarkus to discover it when `iot-testing` is on a consumer's test classpath, the JAR needs `META-INF/jandex.idx`. Without the Jandex Maven plugin in `iot-testing/pom.xml`, the bean exists in the JAR and is completely invisible to CDI. No error, no warning — the injection just fails at test startup. This one is in the garden now.

Chapter 3 starts in Home Assistant territory: REST discovery, WebSocket state subscription, the first real platform. The type system is ready for it.
