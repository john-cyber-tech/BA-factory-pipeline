---
name: traceability-matrix-generator
description: Stitches together whichever of a pipeline's documents currently exist — stakeholder needs, requirements, NFRs, epics, stories, ADRs, architecture — into a single end-to-end traceability matrix, one row per lowest-level traceable item, showing its full lineage from stakeholder need through to architecture component, and explicitly flagging any broken or missing link rather than leaving cells quietly blank. Use this whenever someone asks for a "traceability matrix," "RTM," to "show me the full chain," "can we trace this end to end," "requirements traceability," "what maps to what here," or whenever leadership, audit, or compliance wants a single view proving every requirement is actually accounted for in the design. Works with a PARTIAL chain — most initiatives won't have every pipeline stage done at once — and can be re-run any time more files accumulate.
---

# Traceability Matrix Generator

## Pipeline contract

This is one skill in an eleven-stage BA-to-architecture pipeline. Every stage reads and/or writes a plain Markdown **file** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | nfr-elicitor | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | requirements-to-epics | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | epics-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | epic-to-story-decomposer | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | story-checker | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

`{slug}` is the kebab-case initiative identifier carried in every document's frontmatter and reused across the whole chain — that's how you find all the files belonging to one initiative. This skill isn't a numbered stage — it can run at any point once at least two or three of the files above exist, and it can be re-run later as more accumulate. Don't wait for the full chain to complete before offering it.

**Input contract for this skill:** reads whichever of `{slug}-stakeholder-needs.md`, `{slug}-requirements.md`, `{slug}-nfrs.md`, `{slug}-epics.md`, any `{slug}-epic-*-stories.md`, any `{slug}-adr-*.md`, and `{slug}-architecture.md` are available for the given `{slug}`. Ask the user which files to include if it's unclear which ones exist or which initiative they mean; otherwise work with whatever's supplied without insisting on a complete set.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-traceability-matrix.md`, with the frontmatter block shown in Output Format below. Tell the user the filename and, plainly, which pipeline stages were and weren't represented — so they know whether the matrix is complete or partial.

## Role

A traceability matrix is the artifact leadership, audit, and compliance actually want to see. It answers "can you prove every requirement is accounted for in what's being built" in one glance, without anyone reading eleven separate documents and mentally cross-referencing IDs themselves. That's the whole value proposition: aggregation, not authorship.

Your job here is to assemble and honestly flag gaps, not to author new content or judge quality — the checker skills at stages 3, 6, 8, and 11 already do quality review, and this skill isn't a second copy of that. If a link between two items is missing or broken, note it plainly in this matrix; don't try to fix it, invent it, or improve the underlying document. You're a mirror, not an editor.

## What to build

**Establish the row granularity.** One row per lowest-level traceable item that's actually available — a story if stories exist for that epic, otherwise the epic itself, otherwise the requirement, otherwise the stakeholder need. Don't pad the matrix with duplicate rows by also listing the epic separately when its stories already provide finer-grained rows; roll the epic's identifying columns into each of its story rows instead.

**Trace upward and downward from each row.** For each row, fill in its NEED-ID, REQ-ID, EPIC-ID, STORY-ID, and ADR-ID/COMP-ID — whichever apply and whichever source files exist. Leave a column out of the table entirely if that pipeline stage's file wasn't supplied at all (don't render a column of blanks for a stage that was never run — that's noise, not a finding). But if the file for a stage *does* exist and a specific link is simply missing, mark that cell explicitly (e.g. "No epic found") rather than leaving it blank — a blank cell is ambiguous between "not applicable" and "gap," and this matrix exists specifically so gaps don't hide.

**Flag broken links as findings, not silent blanks.** A requirement with no epic, an epic with no story, a component with no requirement driving it — these are all real findings. List them in a dedicated section separate from the matrix itself, so they aren't easy to miss by skimming a wide table. The matrix shows the shape of what's connected; the findings section is where the disconnections get named.

**Compute simple coverage stats.** Percentage of requirements with at least one epic, percentage of NFRs mapped to a component, percentage of epics with at least one story, and similar — using only the stages actually represented in the input files. Don't compute a stat for a stage that has no file; say it's not yet measurable instead of reporting 0% or omitting it silently.

**Never fabricate a link.** If you can't find a citation in the actual source text connecting two IDs — an epic's stated requirement list, a story's epic reference, an ADR's linked epic — don't infer one just to fill a cell, even when it seems obvious from naming or topic similarity. Mark it unlinked instead and let the finding stand.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: traceability_matrix
schema_version: 1
initiative: <slug>
generated_by: traceability-matrix-generator
source_files: [<list every file actually read>]
date: <today, ISO format>
stages_represented: [<subset of: stakeholder_needs, requirements, nfrs, epics, stories, adr, architecture>]
---

# [Initiative Name] — Traceability Matrix

## Summary

[2-4 sentences: which stages are and aren't represented, and the overall coverage picture — is this a near-complete chain or an early partial one, and what does that mean for how much to trust it.]

## Traceability Matrix

Columns present reflect which stages had a source file; a column is omitted entirely if that stage's file wasn't supplied.

| NEED-ID | REQ-ID | EPIC-ID | STORY-ID | ADR/COMP-ID | Notes |
|---|---|---|---|---|---|
| NEED-003 | REQ-FR-012 | EPIC-002 | STORY-002-04 | ADR-005, COMP-Auth | |
| — | REQ-NFR-004 | EPIC-003 | No story found | — | Gap: NFR not yet decomposed into a story |

## Broken / Missing Links

| Item | Missing Link | Note |
|---|---|---|
| REQ-FR-019 | No epic | Requirement exists but no epic in `{slug}-epics.md` references it |
| EPIC-004 | No story | Epic has no corresponding `{slug}-epic-EPIC-004-stories.md` |
| COMP-Billing | No requirement | Architecture doc names this component with no REQ-ID citing it |

## Coverage Statistics

| Metric | Value |
|---|---|
| Requirements with at least one epic | 11/13 (85%) |
| NFRs mapped to a component | 4/9 (44%) |
| Epics with at least one story | 3/5 (60%) |

[Include only the stats computable from stages actually represented — omit or clearly mark "not yet measurable" for any stage missing entirely.]

## Files Not Yet Available

[List which of the pipeline's files don't exist yet for this slug, so the reader knows the matrix will get more complete once those stages run — e.g. "No `{slug}-adr-*.md` files found — architecture decisions not yet documented."]
```

Save the file as `{slug}-traceability-matrix.md` and tell the user the filename, which stages were represented, and which weren't — so they know at a glance whether they're looking at the full picture or an early slice of it.

## Calibration notes

Work with whatever subset of files is supplied rather than refusing to proceed because the chain is incomplete — a matrix covering only stakeholder needs through epics is still genuinely useful to a team mid-delivery; just say plainly, in the Summary and the Files Not Yet Available section, what isn't represented yet. Don't wait for a "complete" set that may never arrive before this skill produces value.

Don't fabricate a link between two IDs the source documents don't actually connect, even under pressure to make the coverage numbers look better. An honest 60% with a clear gaps list is more useful — and more trustworthy to an auditor — than a fabricated 100%.

If literally none of the pipeline's files exist yet for the given slug, say plainly there's nothing to trace yet and suggest starting the chain with `stakeholder-discovery` (if there's no stakeholder input at all) or `requirements-writer` (if there's already raw input to turn into requirements) — don't produce an empty matrix file just to satisfy the output contract.
