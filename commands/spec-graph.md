---
description: Build an interactive dependency graph from a spec corpus (parse → annotate → render HTML)
argument-hint: "[specs-directory]"
---

Build an interactive spec dependency graph by running the spec-graph pipeline.
This command is a thin orchestrator — the real work lives in three skills, which
you must invoke in order via the Skill tool. Do **not** reimplement their logic
inline; each skill knows how to ask the user for what it needs and adapt to
their conventions.

The user's argument, if any, is the path to their specs directory:

> $ARGUMENTS

## Pipeline

1. **Parse — invoke the `spec-dag` skill.**
   If an argument was provided above, pass it as the candidate specs directory;
   otherwise let the skill ask. Let `spec-dag` do its normal job: confirm the
   directory, list candidate `*.md` files and confirm what counts as a spec,
   confirm the frontmatter convention, and produce the `SPECS` / `EDGES` /
   warnings data. Honor its no-frontmatter handling (offer to generate
   frontmatter) — do not skip it.

2. **Annotate (optional) — offer the `spec-annotations` skill.**
   Ask the user whether they want module / layer / per-section walkthrough
   enrichment. If yes, invoke `spec-annotations` on the parsed corpus. If no,
   continue without annotations — the viewer renders fine without them.

3. **Render — invoke the `spec-graph-viewer` skill.**
   Hand it the DAG (plus any annotations) to produce the self-contained
   interactive HTML, and confirm with the user where to write the output file.

## Notes

- Pass each stage's output to the next; don't re-derive data a prior skill
  already produced.
- If the user only wants a quick text or Mermaid dependency summary, stop after
  step 1 — the `spec-dag` output is useful on its own.
- Surface any warnings (dropped edges, unresolved ids, missing frontmatter) to
  the user at the end.
