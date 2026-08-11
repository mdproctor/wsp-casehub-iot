---
layout: post
title: "Chapter 1 — Why the Type System Matters More Than the Providers"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [iot, api-design, java, builder-pattern]
---

casehub-iot's first chapter delivers the public API module — the types, SPIs, and registry that every consumer and provider depends on. In a semver-from-day-one foundation module, the types you ship first are the types you live with. Getting them wrong costs a major version bump.

The device hierarchy landed on abstract classes. Records are final — a `HomeAssistantLight` can't extend a record `LightDevice` to add `rgbColor()`. Sealed interfaces lose the shared-state constructor chain. So: abstract DeviceEntity base, concrete immutable subtypes, extensible via vendor supplements. Ten types aligned to the Matter Device Type Library — neutral ground between HA's domain classification and OpenHAB's semantic tags.

The builder pattern was the most contested piece. Nine positional constructor parameters for a LightDevice — which `true` is `available`, which is `on`? Self-referential generics: `Builder<T extends DeviceEntity, B extends Builder<T, B>>`. Each subtype's builder inherits named setters from the base. The `T` parameter preserves the product type through the chain. Claude caught during code review that I'd originally used `Builder<B>` — one type parameter — which erased the product type. Generic builder-consuming code couldn't express "a builder that produces T." Binary-breaking to fix after release.

Same review caught that `boolean available` defaults to `false` in Java. Devices default to available; unavailability is the exception. Every provider that forgot `.available(true)` would silently produce offline devices. Fix: `boolean available = true` in the builder field initializer.

The platform's `module-tier-structure` protocol said "No CDI" in Tier 1 api modules. But CdiDeviceRegistry needs `@DefaultBean` from Quarkus Arc, and every consumer is a Quarkus application. I updated the protocol: CDI annotation JARs are inert without a container — they don't force infrastructure configuration. JPA and Mutiny remain excluded. The distinction matters: annotations are metadata, runtimes are obligations.

SPIs are blocking. `discover()` runs at startup; `dispatch()` is a single REST call. Reactive variant deferred — no reactive consumer exists.

CdiDeviceRegistry holds a volatile `Map.copyOf()` reference. Both `refresh()` and `updateDevice()` are synchronized on the same monitor. Without that, concurrent `@ObservesAsync` handlers lose each other's updates, and event/refresh interleaving can revert the entire map to stale state.

The API surface: ten device types, two SPIs, three records, five enums, Temperature value type, CdiDeviceRegistry. What's not here: the providers, the test infrastructure, the bridge. But the contract is done — and getting it right before anyone depends on it is the whole point of Chapter 1.
