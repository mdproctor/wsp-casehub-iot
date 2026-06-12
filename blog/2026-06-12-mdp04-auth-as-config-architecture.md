---
layout: post
title: "Auth as Config Architecture"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-iot]
tags: [openhab, auth, config, cdi, microprofile]
---

The OpenHAB provider shipped in C4 with token auth only — `Authorization: Bearer <token>` hardcoded in the REST client interfaces. Basic auth is the alternative for OpenHAB installs running behind username/password. Straightforward feature addition. XS scale, low complexity.

I expected this to be mechanical. Add an auth-type enum to `OpenHabConfig`, add `username` and `password` fields, switch on the type in the existing `lookupToken()` method. The config interface and both REST clients already had the plumbing.

## The `ConfigProvider.getConfig()` problem

The spec review caught something I hadn't thought about. Both REST client interfaces — `OpenHabRestClient` and `OpenHabSseRestClient` — had identical `lookupToken()` default methods that bypassed the CDI-injected `OpenHabConfig` entirely and went straight to `ConfigProvider.getConfig()` with raw string key lookups. The typed config mapping existed but nothing actually used it for auth. `OpenHabConfig.token()` had zero Java references — only `application.properties` touched it.

My initial design perpetuated this: I restructured `OpenHabConfig` with nested `Auth.Bearer` and `Auth.Basic` groups, then proposed a static utility that read the same data through `ConfigProvider.getConfig()` with new string keys. Two independent read paths for the same config, connected only by string literals. Rename a property in the config interface and the string keys silently break.

The fix was to use `ClientHeadersFactory` — the MicroProfile REST Client 4.0 mechanism for CDI-managed header injection. A single `@ApplicationScoped` bean injects `OpenHabConfig`, validates auth config once in `@PostConstruct`, caches the resolved header, and feeds both REST clients via `@RegisterClientHeaders`. One config read path. The `lookupToken()` default methods and `@ClientHeaderParam` annotations disappear entirely. The REST client interfaces become pure endpoint declarations with no auth logic.

## Anonymous access — the third state

The spec originally said "exactly one of bearer or basic must be configured." Review caught that local OpenHAB installs commonly run without any authentication. The validation rule changed to "at most one" — no auth configured means no `Authorization` header sent. SmallRye constructs the non-Optional `Auth` group with both `Optional<Bearer>` and `Optional<Basic>` empty, and the factory simply doesn't add a header.

The config structure models this cleanly. SmallRye's `Optional<NestedInterface>` handles presence detection: a group is present if any property under its prefix is set, absent otherwise. Partial configuration (e.g. `username` without `password`) is caught by SmallRye's own validation at startup — the factory doesn't need to duplicate that check.
