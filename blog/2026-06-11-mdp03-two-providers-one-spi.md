---
layout: post
title: "C3 and C4 — Two Providers, One SPI"
date: 2026-06-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [openhab, homeassistant, provider, sse, coalescing]
---

The Home Assistant provider went in first. HA's model is flat — one entity ID, one device, state arrives as old/new pairs over WebSocket. The mapper is a pure function: HA state DTO in, `DeviceEntity` out. Everything in the `DeviceProvider` SPI was designed for this case, and it worked without friction.

OpenHAB is a different animal. The data model is two-level: Equipment Groups containing member Point items. A thermostat isn't one entity with attributes — it's a Group tagged `HVAC` with separate items for current temperature, setpoint, and mode. Each with their own semantic tags. Each reporting state changes independently.

That two-level structure forced the interesting design decisions. The mapper stays pure and stateless — it takes an Equipment Group DTO (with inlined members) and produces a single `DeviceEntity`. But the SSE client needs to maintain a state cache: when a single item event arrives, it has to know which Equipment owns that item, update the cached state, reconstruct the full Equipment DTO with all current member values, re-map to a DeviceEntity, and then figure out whether to fire a `StateChangeEvent` immediately or wait.

The "wait" is the coalescing window. OpenHAB fires item events individually — if both the setpoint and mode change simultaneously, those arrive as two separate SSE events milliseconds apart. Without coalescing, consumers would see two partial `StateChangeEvent`s: one where only the setpoint changed, then another where only the mode changed. With a 50ms window, both changes accumulate and consumers get one event reflecting the complete device state transition.

The coalescing implementation taught me something about `ConcurrentHashMap.put()` that I should have known: it returns the old value. The before-snapshot problem — needing to capture what the device looked like before any changes in the current coalescing window — solves itself cleanly with `oldEntity = deviceCache.put(equipmentName, newEntity)`. The put and the snapshot are the same operation. No race, no separate read-then-write.

Four rounds of spec review before implementation caught real bugs. Position inversion — OpenHAB's Rollershutter uses 0=open, 100=closed, the opposite of HA's cover convention. I'd written the mapper without the inversion, which would have silently made every OpenHAB blind report "fully closed" when it was fully open. The spec reviewer also caught that `Control+Switch` on an HVAC equipment is the power toggle, not the mode selector — OpenHAB has no semantic tag for thermostat mode. We ended up matching String items by name containing "mode", which is fragile but honest about the platform gap.

The SSE read-timeout conflict was subtle. Both the REST discovery endpoint and the SSE stream share the same `quarkus-rest-client` infrastructure. A 10-second read-timeout makes sense for discovery and command dispatch — but it would kill the SSE connection in any quiet home where nothing changes for more than 10 seconds. We split into two REST client interfaces with separate config keys: `openhab` (with timeout) and `openhab-sse` (without). The architectural separation between short-lived requests and long-lived streams is cleaner than a single interface trying to serve both.

The HA provider session also closed three trailing issues: `#9` was superseded — the original design planned separate blocking and reactive SPI interfaces, but C3 corrected this by making `DeviceProvider` itself reactive with `Uni<>` returns. `MockDeviceProvider` already implements the reactive SPI. `#10` batched four minor HA code review items. And `parent#211` updated PLATFORM.md with the repo rename and corrected module list.

The position inversion surfaced something worth making explicit. `CoverDevice.position()` now has Javadoc establishing the canonical convention: 0=closed, 100=open. Until now it was only implicit from the HA mapper passing `current_position` through. Any future provider for a platform with the opposite convention will see the contract at the API level, not discover the mismatch through wrong test assertions.
