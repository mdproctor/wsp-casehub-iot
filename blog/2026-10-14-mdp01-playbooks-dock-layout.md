---
layout: post
title: "From Scenarios to Playbooks — and the Dev Mode That Couldn't Start"
date: 2026-10-14
entry_type: note
subtype: diary
projects: [casehubio/iot]
tags: [pages-ui, dockWorkbench, quarkus-dev, cdi]
---

# From Scenarios to Playbooks — and the Dev Mode That Couldn't Start

The IoT console's "Scenarios" page was built with a static `rows(columns([4, 8], ...))` layout — the playbook controller on the left, YAML viewer on the right, fixed proportions. It worked, but it was a dead end. Users couldn't rearrange panels, couldn't add new ones without changing the code, and the page sat visually apart from fsitrading's ops-centre, which had already moved to the dock workbench pattern.

The rename to "Playbooks" was the surface change. The real shift was replacing the static grid with `dockWorkbench()` — a layout that gives each region (centre, left, right, bottom) its own panel zones. The centre holds the playbook controller and YAML viewer in a horizontal split. The three outer regions are empty slots, ready for panels that subsequent issues will wire in. The pattern follows fsitrading's ops-centre exactly: `DockPanelConfig` objects, `storageKey` for layout persistence, `registerPanel()` calls mapping dock keys to web component tags.

Two `registerPanel` calls in `app.ts` connect the dock system to the existing web components:

```typescript
registerPanel("playbook-controller", "pages-playbook-controller");
registerPanel("playbook-yaml-viewer", "pages-playbook-yaml-viewer");
```

The page itself is compact — `dockWorkbench` handles the complexity that `rows` and `columns` used to manage by hand.

## The Dev Mode That Couldn't Start

We tried to start `quarkus:dev` to verify the page visually. It wouldn't start. Not because of our changes — because it had never been startable.

The first layer was CDI: 39 ambiguous `CurrentPrincipal` beans. `TenantScopedPrincipal` from casehub-work and a `MockCurrentPrincipal` producer from casehub-platform both satisfied the same injection point with no priority to break the tie. Neither was annotated `@Alternative`, so `selected-alternatives` couldn't resolve it. We added a `DevCurrentPrincipalProducer` — `@Alternative @Priority(1)`, activated only in the `%dev` profile — that gives the CDI container an unambiguous winner.

The second layer was an empty GraphQL schema. The annotation processor has `generateGraphQL=false`, but `quarkus-smallrye-graphql` arrives as a transitive from `casehub-pages-playbook-runtime`. The extension tries to bootstrap a schema with zero query fields and throws `InvalidSchemaException`. Since GraphQL generation is explicitly disabled, excluding the transitive dependency is the clean fix.

Past those two, there were at least two more layers — Hibernate entity resolution for `SituationEntity` and a missing `drools-wiring-dynamic` module. We filed those as a separate issue rather than chase every pre-existing startup failure in a rename branch.

The CDI and GraphQL fixes are genuine improvements that'll help anyone who tries `quarkus:dev` next. The remaining issues need their own focused effort.
