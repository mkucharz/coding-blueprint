# coding-blueprint

A general coding blueprint for [Emergent Memory](https://emergent.company).
It installs two schema packs into a Memory knowledge-graph project:

- **`product-specs`** — an OpenSpec-style spec workflow: capabilities,
  requirements, scenarios, changes, artifacts, task groups, tasks, releases and
  test suites.
- **`ui-patterns`** — the web-UI pattern catalogue: page recipes, reusable
  components, variant axes, and the cross-cutting rules that govern them.

It also bundles the **`product-specs`** authoring skill.

The goal is the same for both: **keep the source of truth in the graph, not in
a repository file.** Git gives history. The graph gives history, queryability,
a product timeline, traceability from requirement to release, and an inventory
an agent can search before writing code.

---

## What it installs

| Pack | Version | Object types | Relationship types |
|---|---|---|---|
| `product-specs` | 0.2.0 | 9 | 11 |
| `ui-patterns` | 1.2.0 | 4 | 7 |

Plus the `product-specs` skill. No agents. No seed catalogue — the graph is the
catalogue.

---

## Product specs

Product specs are graph objects, not `openspec/` directories.

| OpenSpec artifact | Graph object | Lifecycle |
|---|---|---|
| a capability spec | `Capability` (durable, cumulative) | `active` / `deprecated` |
| a requirement line | `Requirement` (owned by a capability) | `proposed → active → deprecated` |
| a `#### Scenario` | `Scenario` (owned by a requirement) | — |
| a change directory | `Change` + its `Artifact`s | `draft → active → implemented → archived` |
| `tasks.md` checklist | `TaskGroup` → `Task` | `done` flag |
| "shipped" | `Release` | timeline spine |
| the tests that prove it | `TestSuite` | verification |

```
Change ──has_artifact──► Artifact (proposal / design markdown)
  │
  ├──proposes_change (added | modified | removed)──► Capability
  │                                                     │
  │ has_task_group                                      │ has_requirement
  ▼                                                     ▼
TaskGroup ──contains_task──► Task ──addresses──► Requirement ──has_scenario──► Scenario
                                                     │
                              delivered_in ──────────┼──────────► Release
                              verified_by ───────────┴──────────► TestSuite
```

### Object types (9)

| Type | Key prefix | Properties |
|---|---|---|
| `Capability` | `cap:` | `name`, `purpose`, `status` (active/deprecated), `description` |
| `Requirement` | `req:` | `text`, `strength` (SHALL/SHOULD/MAY), `origin` (added/modified/removed), `reason`, `migration`, `status` (proposed/active/deprecated), `description` |
| `Scenario` | `scen:` | `when[]`, `then[]`, `description` |
| `Change` | `chg:` | `name`, `status` (draft/active/implemented/archived), `workflow`, `created`, `archived_at`, `base_ref`, `description` |
| `Artifact` | `art:` | `kind` (proposal/design), `body` (markdown), `order`, `description` |
| `TaskGroup` | `tg:` | `ordinal`, `title`, `description` |
| `Task` | `task:` | `number`, `text`, `done`, `description` |
| `Release` | `rel:` | `version`, `name`, `released_at`, `notes`, `channel`, `description` |
| `TestSuite` | `ts:` | `name`, `status`, `passed_at`, `description` |

### Relationship types (11)

| Relationship | From → To | Meaning |
|---|---|---|
| `has_requirement` | Capability → Requirement | a capability owns a requirement |
| `has_scenario` | Requirement → Scenario | a requirement owns a scenario |
| `depends_on` | Capability → Capability | capability dependency |
| `proposes_change` | Change → Capability | the change adds/modifies/removes the capability (property `operation`) |
| `has_artifact` | Change → Artifact | the change's proposal/design |
| `has_task_group` | Change → TaskGroup | task-list section |
| `contains_task` | TaskGroup → Task | a section contains a task |
| `addresses` | Task → Requirement | the task delivers this requirement |
| `delivered_in` | Requirement, Change → Release | the authoritative delivery edge |
| `verified_by` | Requirement → TestSuite | verification, orthogonal to delivery |

### The workflow loop

**Author → implement → deliver → verify → archive.**

1. **Open a change** — create `chg:<slug>` (`status: draft`).
2. **Attach artifacts** — `art:<slug>:proposal` / `:design` (markdown in `body`).
3. **Declare impact** — `proposes_change` to each affected `Capability` with
   `operation: added|modified|removed` (create the capability if new).
4. **Write the delta** — `Requirement` (`status: proposed`, `origin: …`) +
   `Scenario`s.
5. **Plan the work** — `TaskGroup`s and `Task`s; `addresses` each task to a
   requirement.
6. **Implement** — code branch / PR.
7. **Tick tasks** — `Task.done: true`.
8. **Deliver** — `delivered_in` → `Release` (creates the product timeline).
9. **Verify** — `verified_by` → `TestSuite`.
10. **Archive** — `Change.status: archived`; reconcile the delta:
    `added`/`modified` → `active`; `removed` → `deprecated`.

### Delivery and verification are separate

- **delivered** = a `Release` link exists (`delivered_in`). Answers *when/where
  it shipped*.
- **verified** = a `TestSuite` link exists (`verified_by`). Answers *does it
  work*.

A merged pull request is evidence toward delivery, not proof of it, and not
proof of working. "Delivered but unverified" and "verified but unreleased" are
both valid, queryable states.

### History

The graph keeps what git used to provide:

- `created_at` on every object.
- Object versioning — `version`, `supersedes_id`, `change_summary`, and the
  per-object history endpoint.
- The project journal — every mutation with actor and timestamp.
- Deprecated requirements are retained, never deleted.

---

## UI patterns

`ui-patterns` is the shared web-UI catalogue as graph data. Query it before
writing a page or component: the graph is authoritative for which patterns
exist, when to use them, what they must render, and which shared components
they are built from.

### Object types (4)

| Type | Purpose |
|---|---|
| `UIPattern` | a named, reusable affordance — layout, editing, feedback, widget, nav or data-display recipe |
| `UIComponent` | a reusable element with a stable Go symbol, its props, variant axes, mobile strategy and upstream provenance |
| `UIVariant` | one variant axis of one component — its Go type, allowed values and default |
| `UIRule` | a cross-cutting constraint — PRG contract, htmx ownership, component layers, variants, mobile, upstream-first, role gating, testing |

A `UIPattern` carries the paste-ready `instruction`, the markup `recipe`, the
test contract (`contract_testid`, `contract_aria`, `contract_hx`,
`contract_data`, `contract_other`) and source anchors stamped with
`verified_sha`.

A `UIComponent` carries its Go API (`go_props`), `layer` (L0–L3), `origin`
(upstream / gateway-adapter / gateway-local), `mobile` strategy, a11y
invariants, slots and `upstream_ref`.

### Relationship types (7)

| Relationship | From → To |
|---|---|
| `pattern_governed_by` | UIPattern → UIRule |
| `component_governed_by` | UIComponent → UIRule |
| `pattern_uses_component` | UIPattern → UIComponent |
| `component_has_variant` | UIComponent → UIVariant |
| `component_wraps` | UIComponent → UIComponent (adapter → upstream primitive) |
| `variant_of` | UIPattern → UIPattern |
| `supersedes` | UIPattern → UIPattern |

### Query it

```bash
memory search "list page with filters and per-row actions"      # → a layout recipe
memory query --type UIPattern --filter category=edit            # all editing recipes
memory query --type UIComponent --filter layer=L2               # app composites
```

Over MCP, as an agent does:

```
entity-search "status badge"
  → components.StatusBadge
entity-query type_name=UIVariant filters={component_id: "ui.Badge"} include_properties=true
  → the allowed BadgeIntent values
```

Pass `field_strategy: full` when you need the instruction, recipe or contract;
the default compact strategy returns ids and an empty properties blob.

---

## Install

Validate offline first — no project, no auth:

```bash
memory blueprints validate ./coding-blueprint
memory blueprints inspect ./coding-blueprint
```

Install from GitHub:

```bash
memory blueprints install https://github.com/mkucharz/coding-blueprint --project <project>
```

Pinned ref:

```bash
memory blueprints install https://github.com/mkucharz/coding-blueprint#v1.0.0 --project <project>
```

Or use the authoring loop:

```bash
task validate
task inspect
task dry-run PROJECT=<project>
task install PROJECT=<project>
task upgrade PROJECT=<project>
```

---

## Layout

```
project.yaml
schemas/
  product-specs.yaml      # 9 object types, 11 relationships
  ui-patterns.yaml        # 4 object types, 7 relationships
skills/
  product-specs/SKILL.md  # graph authoring loop
Taskfile.yml              # local validate / inspect / install loop
```

The `schemas/` directory is the preferred blueprint layout (`packs/` is also
accepted). Every pack file declares its own `name`, `version`, `author`,
`license` and `repositoryUrl`.

---

## Source of truth

These packs mirror the installed `memory_coding` graph packs:

- `product-specs` 0.2.0 — 9 types, 11 relationships, 42 properties.
- `ui-patterns` 1.2.0 — 4 types, 7 relationships, 66 properties.

The graph is the live source. This repository is the reviewable copy: the
schema definitions are versioned here, and the published blueprint documents in
the graph carry the same pack definitions. When a pack changes, bump its
version here and in the graph together; schemas are immutable, so a change is a
new version installed with merge semantics, not an edit to a live type.

Presentation metadata (`ui.icon`, `ui.color`, property `title`,
`widget: textarea`, `enum`, `items: {type: …}`) is part of the pack and is what
turns type chips and object forms from bare names and free text into icons,
labels, selects and the right editors.

The `ui-patterns` catalogue itself — patterns, components, variants and rules —
is graph content, not blueprint content. Install the schema from this repo,
then populate the catalogue from its own source. A seed copy here would become
a second catalogue and drift.

---

## Caveats

- **Type-name collisions.** `Scenario` and `TestSuite` collide by name with the
  `code-memory-blueprint` packs. Install `product-specs` into a specs-focused
  project, or confirm the project's type ownership before combining packs.
- **No pull-request type yet.** `product-specs` has no `PullRequest` object or
  PR→spec link. Delivery is release-anchored, not PR-derived.
- **No graph-backed OpenSpec CLI yet.** Once specs live only in the graph,
  file-oriented OpenSpec tooling and repo archive guards no longer apply.
- **`verified_sha` points at the product repository.** UI pattern and component
  anchors are code references, not paths in this repository. Re-grep and
  re-stamp them when the product code moves.

---

## License

MIT
