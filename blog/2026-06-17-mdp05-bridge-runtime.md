---
layout: post
title: "Chapter 5: The Bridge"
date: 2026-06-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [bridge, websocket, jackson, serialization, hexagonal]
---

*Part of a series on [#5 — C5: Bridge Runtime](https://github.com/casehubio/iot/issues/5). Previous: [Two Providers, One SPI](2026-06-11-mdp03-two-providers-one-spi.md).*

I started C5 thinking it was a WebSocket relay. StateChangeEvent goes up, DeviceCommand comes down — a weekend's work. It turned into something better.

The question that changed the direction: what does the cloud-side receiver look like? The issue spec assumed a WebSocket endpoint on "cloud CaseHub." But no such endpoint exists. So I went back to first principles — read through AWS Greengrass, Azure IoT Edge, EdgeX Foundry, Eclipse Ditto's digital twin pattern, MQTT vs WebSocket vs NATS trade-offs. What the industry calls an "IoT edge gateway" is really a pipeline: collect, process, act, export. Not a relay.

But the biggest insight came from staring at casehub-life's Layer 9 spec. `HomeAutomationEventObserver @ObservesAsync StateChangeEvent` — that only works when provider and consumer share a JVM. The bridge's job is to make CDI events flow across a network boundary. And the `DeviceProvider` SPI already abstracts the source.

So: `BridgeDeviceProvider implements DeviceProvider`. The cloud-side module is a library. casehub-life adds it as a dependency. Remote devices look local. `HomeAutomationEventObserver` fires the same whether the event came from a local HA instance or a bridge agent across the internet. The hexagonal architecture we built in C1 pays off in C5 — the SPI was designed for exactly this.

Three design problems made this harder than the architecture:

**Polymorphic serialization.** DeviceEntity is an abstract class with 16 concrete subtypes. `deviceClass` can't discriminate — both `ThermostatDevice` and `HomeAssistantThermostat` have `deviceClass = THERMOSTAT`. I spent time on a `DeserializationProblemHandler` approach before discovering that `handleUnknownTypeId()` has no access to sibling JSON fields. Verified against the Jackson 2.21.1 source — the method signature takes only the type ID string. The fix: compound type IDs. `"THERMOSTAT:HomeAssistantThermostat"` embeds the fallback in the ID itself. The `DeviceTypeIdResolver` splits on `:`, tries the full match, falls back to the DeviceClass prefix. Unknown supplements degrade gracefully to their common parent.

A second Jackson surprise: `context.registerSubtypes()` feeds `TypeNameIdResolver`, not a custom resolver. Vendor modules calling `registerSubtypes()` silently fail to register — no error, no warning, complete loss of type fidelity. The fix: vendor modules call `DeviceTypeIdResolver.registerType()` directly.

**Device ID collision.** `CdiDeviceRegistry` keys by `deviceId()` alone. Two HA installations both have `light.kitchen`. Multi-site deployments — a property company managing 50 rentals — would silently overwrite devices. Server-side namespacing via Jackson tree copy: `valueToTree()` → modify `deviceId` → `treeToValue()`. The Jackson infrastructure we added for serialization doubles as a polymorphic copy mechanism, sidestepping the `toBuilder()` type-slicing problem on the four supplementable device types.

**Event replay semantics.** The original design had an in-memory buffer for disconnection. Claude caught the flaw during review: replaying a presence event from 30 seconds ago triggers a security case for a past arrival, while the current state shows no presence. Ghost automations. The fix is simpler than the buffer: on reconnect, send a state snapshot from `discover()`. Cloud gets current truth. No buffer, no replay, no ghosts.

The bridge ships as two modules: `casehub-iot-bridge` (standalone Quarkus app, the local agent) and `casehub-iot-bridge-server` (library, the cloud-side `BridgeDeviceProvider`). Between them sits a sealed `BridgeMessage` interface — six record variants, exhaustive `switch`, no default branches. A `BridgeEventFilter` SPI lets consumers plug in CDI-discovered filter functions before cloud relay: throttling, privacy, bandwidth management.

Hybrid deployment — local Drools for latency-sensitive reactions, cloud for orchestration — isn't a bridge feature. It's a deployment topology. Add your application-tier JAR to the bridge's classpath; your `@ObservesAsync` beans fire locally alongside the relay. The bridge stays domain-logic-free.

C1 through C5 are done. The casehub-iot journey is complete — typed device hierarchy, two platform providers, test harness, and a bridge that makes the whole thing work over a network without anyone noticing.