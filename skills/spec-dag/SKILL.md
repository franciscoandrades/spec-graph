---
name: spec-dag
description: >
  Use when the user wants to parse a corpus of spec markdown files into a
  dependency graph — trigger phrases include "parse my specs into a dependency
  graph", "what depends on what", "build a DAG of specs", "show how my specs
  relate", "which spec depends on which", or "map out spec dependencies". This
  skill produces structured data (SPECS array, EDGES array, warnings). It does
  NOT render HTML (use spec-graph-viewer for that) and does NOT add
  module/layer/walkthrough enrichment (use spec-annotations for that).
---

## spec-dag — Parse specs into a dependency DAG

This is an **agent task, not software.** Read the files and extract the data
inline, in one turn. Do NOT write a Python package, CLI, or test suite unless
the user explicitly requests a re-runnable tool.

---

### Step 1 — Locate the specs directory

Ask the user where their spec files live. Do not assume a path. Common
candidates to suggest: `docs/specs/`, `docs/`, `specs/` — but confirm before
proceeding.

---

### Step 1b — List candidate files and confirm what counts as a spec

Once the directory is known, glob it for `*.md` files and **present the full
list to the user** before reading or parsing anything. Do not assume every
`*.md` file is a spec.

For each candidate, show enough to let the user decide: the filename, and (if
cheaply available) its `title`/first H1 and `type` frontmatter field. Then ask
the user to confirm the selection. Useful prompts:

- "Here are the N markdown files I found — which of these should be treated as
  specs?"
- Offer obvious filters when the corpus is mixed: e.g. "include only files with
  `type: spec`", "exclude `README.md` / `CHANGELOG.md` / templates", or "include
  everything in `docs/specs/` but nothing in `docs/notes/`."

Proceed only with the confirmed set. If the user gives a rule (e.g. "anything
with `type: spec`"), apply it but still report the resulting list so they can
catch surprises. Re-confirm if the glob surfaces files in unexpected
subdirectories.

---

### Step 2 — Confirm the frontmatter and description conventions

The skill must adapt to the user's schema. Present the example below as one
possible format and ask the user to confirm or describe their own convention
before reading any files.

**Example format (one possible schema — confirm with user):**

```yaml
---
title: My spec
date: 2026-01-15
status: Approved
type: spec
related:
  - prerequisite-spec-id
  - id: another-spec-id
    because: "needed for the data model section"
---

> **Purpose / Status.** One paragraph describing what this spec does and why.
> Continuation lines belong to the same blockquote.

## §1 Motivation
…
```

Key conventions to clarify with the user:
- Which frontmatter field lists dependencies? (In the example: `related:`)
- Is each dependency entry a bare string id, an object `{id, because}`, or
  something else?
- Which field (if any) marks the file type so non-spec files (ADRs, plans,
  notes) can be skipped? (In the example: `type: spec`)
- How is the spec's summary expressed? (In the example: the first `>` blockquote
  block, opened with a bold label like `**Purpose / Status.**`)

If the user's format differs significantly from the example, adapt all
extraction steps below accordingly.

---

### Step 3 — Read every `*.md` file and extract per-spec fields

For each file found, extract:

| Field | Source |
|-------|--------|
| `id` | Filename without `.md` extension |
| `title` | Frontmatter `title:`; else first `# H1` in the body; else the `id` |
| `full_title` | Same as `title` (reserved for side-panel display) |
| `date` | Frontmatter `date:` or `""` |
| `status` | Frontmatter `status:` or `""` |
| `related` | Frontmatter dependency list (name confirmed in Step 2). Each entry is a bare id string OR an object with id + reason. Normalize every id: strip `.md` extension and any `./` prefix. Empty list if absent. |
| `purpose` | The first description blockquote (per the convention confirmed in Step 2). Strip the leading `> ` from each line, join continuation lines into one paragraph, strip any bold label and leading separator (em-dash or similar). Preserve double-spaced paragraphs as blank lines. |
| `is_orphan` | Initialize `false`; set in Step 5. |
| `path` | Relative path from the repo/target root to the spec file. |

**Edge cases — handle explicitly:**

- **No frontmatter at all:** Do NOT silently guess, and do NOT just interrogate
  the user field by field. **Offer to generate the frontmatter for them by
  reading the spec.** Read the file's prose and propose a complete frontmatter
  block (using the schema confirmed in Step 2):
  - `title` from the first H1 or the document's evident subject.
  - `related` inferred from the prose — any references to other specs by name,
    id, or filename, and any "depends on / builds on / requires" language. Only
    list dependencies you can point to in the text; never invent one.
  - `status`/`type` and any other schema fields if the prose makes them clear;
    otherwise leave them for the user to fill.

  Present the proposed block to the user and ask them to confirm or correct it.
  On confirmation, **write the frontmatter back into the file** so the corpus is
  consistent, and proceed with those values. If the user declines the offer
  entirely, fall back to `title` from first H1, `related: []`, and emit a WARNING
  flagging the file for inspection.
- **Type field present and not `spec`** (or whatever the user's equivalent is):
  Skip the file entirely — it is an ADR, plan, or note, not a spec.
- **Malformed YAML:** STOP. Tell the user which file failed and what the parser
  error is. Do not silently skip.

---

### Step 4 — Build the edges array

For each spec, create one edge per entry in its dependency list:

```
{ from: <this spec id>, to: <dependency id>, because: <string|null> }
```

Semantics: `from` = successor (the spec that depends), `to` = predecessor (the
spec being depended on). Downstream renderers rely on this direction — be
precise.

- Bare-string entry → `because: null`
- Object entry `{id, because}` → copy `because` verbatim
- Optionally, if the successor's `purpose` text clearly states the reason for
  the dependency, lift it as `because`. Never invent one.
- If a dependency id matches no parsed spec → DROP the edge and emit a WARNING
  naming both the referencing spec and the missing id.

**Cycle handling:** If the dependency list would introduce a cycle, keep the DAG
acyclic by dropping the edge that closes the cycle. Emit a WARNING naming the
dropped edge (both endpoints). Continue processing; do not abort.

---

### Step 5 — Mark orphans

After edge resolution: a spec with zero incoming edges AND zero outgoing edges
is an orphan. Set `is_orphan: true` on it. This includes specs that had no
frontmatter and specs whose only edges were dropped.

---

### Step 6 — Compute topological ordinals

Topologically sort the resolved edge set. Tie-break lexicographically by `id`.
Order: roots first (specs with no predecessors), then their successors in
dependency order, orphans last. Assign ordinal 1, 2, 3, … to each spec. Store
as `ordinal` on each spec record.

---

### Step 7 — Report the structured result

Output:

1. **SPECS** — array of spec records (all fields above including `ordinal`).
2. **EDGES** — array of edge records (`from`, `to`, `because`).
3. **Warnings** — list of all warnings emitted (missing frontmatter, dropped
   edges, dropped cycle-closing edges, unresolved ids).

This data is useful on its own: if the user just wants a quick dependency
summary, render a compact text list or a Mermaid flowchart directly from it.

If the user wants an interactive HTML graph, hand this data to the
**spec-graph-viewer** skill. For module/layer/section enrichment of the nodes,
hand it to the **spec-annotations** skill.

---

### Critical rules

- **Ask, don't assume.** If the specs directory, frontmatter schema, or
  description convention is unclear at any point, ask the user rather than
  guessing.
- **Agent task, not a package.** Extract data inline. Do not write files,
  create a Python project, or scaffold a CLI unless the user explicitly requests
  a re-runnable tool.
- **Edge direction is successor → predecessor.** This is the canonical direction;
  every downstream consumer (viewer, annotations) depends on it.
- **No hardcoded paths or domain assumptions.** Every project is different;
  nothing in this skill is specific to any single codebase or naming convention.

---

### Next steps

- Interactive HTML graph of the DAG → **spec-graph-viewer** skill
- Annotate nodes with module/layer/walkthrough metadata → **spec-annotations** skill
