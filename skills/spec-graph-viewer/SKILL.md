---
name: spec-graph-viewer
description: >
  Use this skill when the user wants to visualize a spec corpus as an
  interactive dependency-graph HTML page — phrases like "build the spec graph
  viewer", "generate the spec graph", "render the spec DAG", or "show me the
  spec graph". Requires a parsed DAG from spec-dag; optional enrichment from
  spec-annotations. Produces a single self-contained HTML file — no build step,
  no separate assets.
---

# spec-graph-viewer

Render a parsed spec DAG into one self-contained interactive HTML file by
substituting data into the bundled template.

## Prerequisites

- **spec-dag** (required) — must have already run and produced `SPECS`,
  `EDGES`, and `warnings`. If not available, tell the user to run `spec-dag`
  first.
- **spec-annotations** (optional) — produces `MODULES`, `LAYERS`, and per-spec
  `walkthrough` sections. If not available, pass empty arrays; the filter rails
  and walkthrough tabs are simply disabled.

Ask the user where to write the output HTML file. Suggest `docs/spec-graph.html`
as a default. Create the parent directory if it does not yet exist.

---

## Procedure

### 1. Collect the data

Gather the following from the sibling skills (or from context if already in
scope):

| Variable | Source | Fallback |
|---|---|---|
| `SPECS` | spec-dag | — (required) |
| `EDGES` | spec-dag | — (required) |
| `warnings` | spec-dag | `[]` |
| `MODULES` | spec-annotations | `[]` |
| `LAYERS` | spec-annotations | `[]` |

#### SPECS — per-entry shape

```json
{
  "id":         "spec-01",
  "ordinal":    1,
  "title":      "short title",
  "full_title": "full document title",
  "date":       "2024-01-15",
  "status":     "draft",
  "purpose":    "one-sentence purpose extracted from frontmatter or §0 blockquote",
  "path":       "docs/spec-01.md",
  "is_orphan":  false,
  "modules":    ["auth", "api"],
  "layers":     ["L3"],
  "content":    "optional — full markdown body",
  "walkthrough": []
}
```

- `path` is the relative path used by the "View source" link in the panel.
- `is_orphan: true` for any spec with no parseable frontmatter; include it with
  a warning rather than silently dropping it.
- `modules` and `layers` are arrays of ids; use `[]` when no annotations are
  available.
- `walkthrough` is an array of `{ "title": "…", "modules": ["…"], "body": "…" }`
  objects (one per section); use `[]` when not provided by spec-annotations.

#### EDGES — per-entry shape

```json
{ "from": "spec-02", "to": "spec-01", "because": "optional rationale or null" }
```

**Direction convention:** emit edges as **successor → predecessor** (`from` is
the downstream spec that depends on `to`). The template flips source/target on
render so arrows visually flow predecessor → successor. Reversing this points
every arrow the wrong way.

#### MODULES — per-entry shape

```json
{
  "id":          "auth",
  "name":        "Authentication",
  "type":        "component",
  "description": "Handles user identity and session management."
}
```

`type` must be one of: `constraint`, `component`, `endpoint`, `entity`,
`library`, `reference`. These are overridable defaults; adapt to the project's
vocabulary.

#### LAYERS — per-entry shape

```json
{ "id": "L3", "letter": "C", "name": "Application", "description": "Core business logic." }
```

Pass `[]` to disable layer rendering entirely.

---

### 2. Read the template

Read the file at:

```
${CLAUDE_PLUGIN_ROOT}/skills/spec-graph-viewer/templates/spec-graph.html
```

Use `${CLAUDE_PLUGIN_ROOT}` literally — it is resolved at runtime to the
plugin root, keeping the skill portable across installations.

The template is complete and must not be modified. It contains exactly four
placeholder tokens to substitute:

| Placeholder | Replace with |
|---|---|
| `__SPECS_JSON__` | SPECS array as JSON |
| `__EDGES_JSON__` | EDGES array as JSON |
| `__MODULES_JSON__` | MODULES array as JSON (`[]` if none) |
| `__LAYERS_JSON__` | LAYERS array as JSON (`[]` if none) |

---

### 3. Serialize and embed — two mandatory escape steps

Both steps are required. Both have caused silent breakage before.

**Step A — escape `</` inside JSON strings.**
Before substituting, replace every `</` with `<\/` in the serialized JSON
string. This prevents the HTML parser from closing the `<script>` block early
when a spec body contains tags like `</span>` or `</div>`. `\/` is a valid
JSON escape for `/`; the data is semantically unchanged.

```python
json_str = json.dumps(value, ensure_ascii=False)
json_str = json_str.replace("</", "<\\/")
```

**Step B — use a lambda (not a string) as the regex replacement.**
Python's `re.sub` / `re.subn` interprets backslash sequences (`\n`, `\1`,
`\g<name>`, …) in replacement strings. Feed the replacement via a callback so
the content is treated verbatim:

```python
import re

# WRONG — replacement string is parsed for backslash sequences:
html = re.sub(r"__SPECS_JSON__", json_str, html)

# RIGHT — lambda returns replacement verbatim:
html = re.sub(r"__SPECS_JSON__", lambda _: json_str, html)
```

Apply both steps to each of the four placeholders.

---

### 4. Write the output file

Write the substituted HTML string to the user's chosen output path. Create the
parent directory first if it does not exist.

---

### 5. Report

After writing, output:

1. **Path written** — absolute path of the generated file.
2. **Stats** — number of specs, edges, and any warning count.
3. **Warnings** — list the warning messages (orphan specs, missing frontmatter,
   unresolved references, etc.).
4. **How to open** — `xdg-open <file>` on Linux; `open <file>` on macOS.

---

## What the template provides (do not modify it)

- **Graph engine:** Cytoscape.js via unpkg CDN, `breadthfirst` layout with
  `directed: true`. No layout plugins (cytoscape-dagre is intentionally
  excluded — its unpkg URL is unstable).
- **Markdown rendering:** marked via unpkg CDN, used in the side panel.
- These two CDN scripts are the only external dependencies.
- **Dark theme**, fully inline CSS, click a node or edge to open a right-side
  panel with Overview and optional Walkthrough tabs.
- **Left panel filter rails:** a Components rail (from `LAYERS`) above a
  Modules rail (from `MODULES`). Both rails are clickable chips, AND-combined —
  a spec stays highlighted only when it matches every active filter. Single-
  select within each rail; clicking the active chip clears that dimension.
  Both rails are hidden when their data array is empty.

Leave all of this structure as-is; only substitute the four data markers.

---

## Critical rules

- One self-contained HTML file — no separate CSS, JS, or asset files.
- No Python package, CLI wrapper, or test suite — this is an agent task: read,
  substitute, write.
- Never assume an output path — ask the user; suggest `docs/spec-graph.html`.
- No hardcoded corpus, module names, layer names, or project names in this
  skill. Every example above is an illustrative placeholder.
- Any spec with unparseable frontmatter is the generic "orphan" case: include
  it with `is_orphan: true` and emit a warning. Do not drop it silently.
- Edge direction is **always** successor → predecessor in the data array;
  the template handles the visual flip.

---

## Cross-references

- **spec-dag** — required upstream skill; parses the corpus and produces
  `SPECS`, `EDGES`, and `warnings`.
- **spec-annotations** — optional upstream skill; produces `MODULES`, `LAYERS`,
  and per-spec `walkthrough` data.
