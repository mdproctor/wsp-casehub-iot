---
layout: post
title: "CaseHub IoT — Three Walls Between Config and Runtime"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [quarkus, rest-client, config, discovery, docker, audit]
---

# CaseHub IoT — Three Walls Between Config and Runtime

**Date:** 2026-06-26
**Type:** phase-update

---

## What I set out to do

Three operational features for the bridge: Docker Compose deployment for Raspberry Pi, mDNS/SSDP auto-discovery of Home Assistant and OpenHAB, and a server-side audit event log. Straightforward scope — deploy it, discover providers automatically, record what flows through.

The interesting part turned out to be what stood between the current codebase and those features.

## Wall 1: SmallRye resolves expressions before anything else

The provider modules bind REST client URLs via property expressions in `application.properties`:

```properties
quarkus.rest-client."homeassistant".url=${casehub.iot.homeassistant.url}
```

Making `url()` return `Optional<String>` on the `@ConfigMapping` interface felt like the obvious fix for disabled providers. It is not. SmallRye Config resolves `${...}` expressions at config startup — before ConfigMapping interfaces exist, before CDI starts, before any bean lifecycle runs. The expression layer and the ConfigMapping layer are independent. If the property doesn't exist, the expression fails regardless of what the interface says.

A dummy default (`${casehub.iot.homeassistant.url:http://unused.local}`) prevents that crash but creates the second wall.

## Wall 2: REST client base URLs are immutable

`@RestClient` beans get their base URL baked in at config resolution time. It cannot be changed after creation. So even if mDNS discovers `http://192.168.1.50:8123` during `@PostConstruct`, the REST client is already pointing at whatever the config said — the dummy default, or nothing.

The WebSocket clients don't have this problem. `HomeAssistantWebSocketClient.connect()` resolves the URL programmatically: `connectorProvider.get().baseUri(URI.create(url)).connect()`. The URL is passed at connect time, not at bean creation time. The REST clients had no such escape hatch.

The fix: switch from `@RegisterRestClient` and CDI injection to programmatic `RestClientBuilder`. The interface keeps its JAX-RS annotations (`@GET`, `@Path`) — they work identically with `RestClientBuilder`. The `@RegisterRestClient` annotation, the property expression, and the CDI injection point are all removed. The provider creates the client in its own startup with the URL it actually wants, whether from config or from discovery.

## Wall 3: ClientHeadersFactory isn't what you think it is

OpenHAB's auth was handled by `OpenHabAuthHeadersFactory implements ClientHeadersFactory`, registered via `@RegisterClientHeaders`. When I moved to `RestClientBuilder`, the natural refactor was `.register(authHeadersFactory)`. It compiles. It runs. No error. No warning. Requests go out without the Authorization header. The API returns 401.

`ClientHeadersFactory` is a MicroProfile REST Client extension interface. `RestClientBuilder.register()` inherits from `Configurable<>`, which only activates JAX-RS provider types — `ClientRequestFilter`, `MessageBodyReader`, etc. `ClientHeadersFactory` is not one of them. The `.register()` call silently accepts any object and silently ignores anything it doesn't recognise.

The fix: `ClientRequestFilter`. Same auth logic, correct interface. The factory class is deleted.

## The tenancyId that shouldn't have been three

While wiring up the Docker image (both provider modules on one classpath), I found that `BridgeAgentConfig`, `HomeAssistantConfig`, and `OpenHabConfig` each define their own `tenancyId()`. In a bridge deployment, all three must match — the bridge stamps wire messages with its tenancyId, the providers stamp entities with theirs. Three properties that must always hold the same value is not configuration, it's a divergence bug.

Consolidated to a single `casehub.iot.tenancy-id` root property. Every consumer injects it via `@ConfigProperty`. One env var, zero divergence risk.

## What actually shipped

Provider activation via `@LookupIfProperty` — disabled providers are invisible to `Instance<DeviceProvider>`, not zombie beans returning empty. Auto-discovery via JmDNS for both HA (`_home-assistant._tcp`) and OpenHAB (`_openhab-server._tcp` with SSL preference, SSDP fallback). Audit events as CDI events — not an SPI, because the dual-trail pattern (operational logging + optional compliance ledger) requires multiple simultaneous observers, and CDI picks one SPI implementation. Dockerfile, Compose, deployment guide, multi-arch CI for ARM64.

The three walls took more design time than the three features. Every one of them was invisible until the existing assumptions — required config properties, CDI-injected REST clients, MicroProfile's type hierarchy — collided with a runtime that needs to make decisions after startup.