---
name: spec-annotations
description: >
  Use when the user wants to semantically enrich a spec corpus that has already
  been parsed into a DAG (by the spec-dag skill): tag which specs touch which
  cross-cutting modules (shared constraints, components, entities, libraries, or
  reference material), assign architectural layer/component badges to each spec,
  and build a per-section walkthrough array that makes long specs scannable.
  Also use when the user wants to update or audit annotations after new specs
  have been added, or when they ask "which specs all touch the same X?"
  This skill does NOT parse the corpus (spec-dag) and does NOT render HTML
  (spec-graph-viewer).
---

## Overview

Annotations are OPTIONAL — the spec-graph viewer renders a graph without them.
Three annotation layers are produced:

1. **Modules** — a named ontology of cross-cutting elements shared by ≥2 specs.
2. **Layers / components** — a coarse taxonomy of WHERE in the architecture each
   spec lands.
3. **Walkthrough** — a per-section semantic index for every spec.

Work in this order: modules → layers → walkthrough.

---

## 0. Source-of-truth priority (applies to both modules and layers)

Before inferring anything, check for existing taxonomy files:

1. **`modules.yaml` / `layers.yaml`** co-located with the spec corpus —
   authoritative; use as-is, do not invent ids that conflict.
2. **`modules:` / `layers:` frontmatter keys** inside individual spec files —
   honour ids already defined in the ontology file; unknown ids warn and drop.
3. **Agent inference** — read every spec, find recurring elements. Only fall
   back to this when neither (1) nor (2) exists.

If the user has no taxonomy files and wants annotations, ASK them to either:
- Create a `modules.yaml` / `layers.yaml` with their own taxonomy, OR
- Confirm that you will infer a taxonomy by reading the corpus.

Never silently impose a taxonomy.

---

## 1. Modules

### What is a module?

A module is any element that recurs across **two or more** specs and is worth
surfacing as a cross-cutting concern: a shared constraint, a reusable component,
an API endpoint or contract, a data entity, an external library, or a piece of
reference material. Modules let the user answer "which specs all touch the same
X?"

### Schema

```yaml
# modules.yaml (authoritative ontology file)
- id: <kebab-case>          # unique, stable identifier
  name: <≤40 characters>    # display name
  type: <see palette below> # controls colour in the viewer
  description: >            # 1–3 sentences
    ...
```

**Default type palette** (these six are coloured by the viewer; new types are
allowed but rendered uncoloured):

| type | examples |
|------|---------|
| `constraint` | a project-scope rule, a shared design-system constraint, an auth policy |
| `component` | a reusable UI component, a shared service |
| `endpoint` | an API route or integration surface |
| `entity` | a data model, a domain object |
| `library` | a third-party dependency cited across specs |
| `reference` | external standards, architecture documents, or normative references |

### Inference procedure (when no modules.yaml exists)

1. Read every spec in the corpus (strip frontmatter before analysing).
2. List all recurring named elements — constraints referenced in multiple specs,
   components described or consumed in multiple specs, entities appearing in
   multiple data flows, libraries imported in multiple specs, shared reference
   documents.
3. Keep only elements touching **≥2 specs**. A one-spec element is noise.
4. Cap the total at ~15–20 modules. Prefer fewer, higher-signal modules over a
   long flat list.
5. Assign a `type` and write a 1–3 sentence `description` for each.
6. Emit as a `MODULES` array (or write `modules.yaml` if the user approves).
7. Add `modules: [<id>, …]` to each spec's frontmatter — include every module
   the spec meaningfully touches.

### Updating an existing graph (new specs added)

- Audit each new spec against existing MODULES; tag all that apply (common
  miss: omitting a clearly applicable module because the new spec doesn't name
  it verbatim).
- Look for NEW recurring elements shared by ≥2 of the new specs.
- **Backfill** old specs when a newly surfaced module also applies to them.
- Drop modules that have fallen below the ≥2 threshold.
- Goal: every spec's module list tells a coherent story. A spec with only one
  module is a smell unless it genuinely touches only one cross-cutting thing.

---

## 2. Layers / components

### What is a layer?

A layer describes WHERE in the architecture a spec primarily lands — a coarse
architectural slice, not what the spec is about. One spec can touch multiple
layers.

### Schema

```yaml
# layers.yaml (authoritative taxonomy file)
- id: <kebab-case>
  letter: <single uppercase char>   # shown as node badge and filter chip
  name: <display name>
  description: >
    ...
```

### Default example set (WEB STACK — override for your domain)

> **This is an example only.** Define your own layer taxonomy for your domain
> (e.g. mobile, ingestion, analytics, embedded, infra). The web-stack set below
> ships as a starting point to confirm or replace.

| id | letter | name |
|----|--------|------|
| `frontend` | F | Frontend |
| `api` | A | API / Backend |
| `processing` | P | Processing / Workers |
| `database` | D | Database |
| `storage` | S | Storage |

If the user has no `layers.yaml` and wants layer annotations, show the default
set and ask: "Does this match your stack, or should we define a custom layer
taxonomy first?"

### Inference procedure

1. For each spec, identify which layers it primarily touches (creates code,
   defines contracts, or specifies behaviour for).
2. A spec that describes only a cross-cutting policy or a project-scope
   constraint may legitimately touch all layers — tag it with all relevant ids.
3. Add `layers: [<id>, …]` to each spec's frontmatter.

---

## 3. Per-section walkthrough

### Purpose

Makes long specs readable: section structure at a glance, each section labeled
with the module(s) it is *about*, expandable in the viewer.

### Data shape

```js
// Viewer attaches walkthroughs via a top-level keyed object:
SPEC_WALKTHROUGHS = {
  "<spec-id>": [
    { title: "<section heading>", modules: ["id1", "id2"], body: "<markdown>" },
    ...
  ],
  ...
}
```

Keep each spec's walkthrough payload separable and regenerable independently.

### Procedure per spec

1. Read the file; strip frontmatter.
2. Split at `## ` headers. Content before the first `## ` is the `(intro)`
   section. `### ` subsections stay within their parent `## ` section — do not
   split on them.
3. For each section, pick **0–3 modules** it is *about* (most important first;
   stop at 3).
4. Pure-structure sections (reading guides, indexes, bare appendices with no
   substantive prose) get `modules: []`.
5. Emit `{ title, modules: ["id1", …], body }` where `body` is the section's
   full markdown (including any `### ` subsections).

### Calibration heuristics

Apply these as semantic judgement, not keyword rules:

- **Motivation / Why sections** — usually the cross-cutting constraint that
  motivates the work, plus the concrete thing being created.
- **Reference-material sections** — tag with the external references or
  normative references modules, not the constraint modules.
- **Architecture / Design sections** — tag with the architectural or
  infrastructure modules the section specifies.
- **Component / Implementation sections** — tag with the specific
  component, entity, or library the section creates or modifies.
- **Considerations / Open questions / Acceptance criteria** — usually a
  scope constraint only, or empty.
- **Decision / Consequences (ADR-style)** — same modules as Context, sharper.

### Anti-patterns (warn if you catch yourself doing these)

- **Keyword-counting** — tagging a section because it mentions a term, not
  because the section is *about* that module.
- **Universal scope module on every section** — uninformative; reserve the
  scope/constraint module for sections that are genuinely gated by it.
- **Code-block tag bleed** — tagging a section with whatever the code mentions
  rather than what the surrounding prose is about.
- **Skipping long specs** — length is not a reason to skip; every spec gets a
  walkthrough.

### Manual override placeholders

Authors may embed these inside spec section bodies; the viewer renders them:

- `[[module:<id>]]` — inline module chip; clicking it filters the graph.
- `[[layer:<id>]]` — inline layer chip.
- `[[spec:<id>]]` — cross-reference link to another spec.

Unknown ids fall back to plain text in the viewer.

Note: section bodies may contain raw HTML or JSX — that is fine; the
**spec-graph-viewer** skill handles safe embedding.

---

## 4. Output

Report the following after each annotation run:

- `MODULES` array (id, name, type, description) with per-spec module tags.
- `LAYERS` array (id, letter, name, description) with per-spec layer tags.
- Per-spec `walkthrough` data (title + modules per section).

If writing back to files: update each spec's frontmatter `modules:` and
`layers:` keys, and write the walkthrough payload to a `spec-walkthroughs.js`
(or `.json`) file in the graph output directory.

---

## Cross-references

- Operates on a corpus already parsed by the **spec-dag** skill.
- Annotation output feeds the **spec-graph-viewer** skill for rendering.
