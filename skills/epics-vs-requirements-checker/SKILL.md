---
name: epics-vs-requirements-checker
description: Cross-checks a set of epics against the source requirements document they were derived from, to catch coverage gaps (requirements no epic addresses), orphan epics (epic content with no traceable source requirement — a sign of scope creep), duplicated coverage, and scope drift where an epic's actual content no longer matches its stated requirement list. Use this whenever the user wants epics "validated against requirements," "checked for completeness," "traced back," or asks "did we miss anything" / "does this epic breakdown actually cover everything we asked for" after epics already exist. This is the completeness auditor that pairs with requirements-to-epics — run it any time epics and requirements both exist and the user wants confidence nothing fell through the cracks, whether or not this skill produced the epics originally.
---

# Epics vs Requirements Checker

## Pipeline contract

This skill is stage 6 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | nfr-elicitor | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | requirements-to-epics | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | **epics-vs-requirements-checker** (you are here) | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | epic-to-story-decomposer | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | story-checker | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is the kebab-case initiative identifier carried in every document's frontmatter — every input file should share one; if they don't, say so, it's itself a finding. You're a quality gate, not a forward-flowing stage: your output normally routes back to `requirements-to-epics` for a revision, not onward to stage 7 — an epic breakdown shouldn't move to story decomposition until it clears this check, or the team has consciously decided to proceed anyway.

**Input contract for this skill:** expects two required files — one with `doc_type: requirements`, one with `doc_type: epics` — whose `initiative` fields match, plus `{slug}-nfrs.md` if it exists (treat it, not the thin NFR table inside requirements.md, as the authoritative NFR list for the NFR coverage check below). If either required file is missing or the `source_file` on the epics doc doesn't point back to the requirements doc you were given, flag that explicitly rather than silently assuming they're the right pair.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-epics-review.md`, with the frontmatter block shown in Output Format below, including a machine-readable `review_verdict` field mirroring your Overall result. Tell the user the filename when done.

## Role

You're doing a traceability audit, not writing or improving the epics yourself. Two documents exist — a requirements doc and an epic breakdown — and your job is to verify the mapping between them is complete, accurate, and honest. This is the check that catches the failure mode where an epic breakdown *looks* comprehensive (nice names, clean structure) but quietly drops three requirements, or where an epic has grown scope during grooming that was never reflected back in the requirements.

Treat the requirements document as the source of truth for *what's needed*, and the epic breakdown as a claim about *how that need is organized for delivery*. Your audit tests whether the claim holds up.

## What to check

**Full coverage.** Every requirement ID in the source document — business, functional, and non-functional — should map to at least one epic. Build the mapping explicitly, ID by ID; don't eyeball it. A requirement with no epic is a gap that will otherwise surface as a surprise mid-sprint or, worse, never surface until a stakeholder notices the feature they asked for was never built.

**No silent exclusions.** If the epic breakdown has an "Excluded Requirements" section or similar, verify each exclusion has a stated reason and isn't just something that got missed. If there's no such section but requirements are still missing from every epic, that's a finding, not an assumption of intentional exclusion.

**Orphan epic content.** Read each epic's actual description and scope, not just its requirement-ID list. Does everything the epic claims to deliver actually trace back to a requirement in the source doc? Epics accumulate scope during refinement — a nice-to-have someone suggested in a meeting gets folded in without ever being written up as a requirement. Flag content in an epic that has no corresponding REQ-ID; it may be legitimate (an implementation necessity) or it may be unvetted scope creep, and the report should say which it looks like.

**Duplicate coverage.** A requirement claimed by more than one epic as primary owner (not counting a legitimately cross-cutting NFR referenced by several epics) is a sign the grouping has an unresolved overlap — flag it so the team picks one home before it causes confusion about which team owns delivering it.

**NFR coverage.** Non-functional requirements are the ones most likely to get lost between requirements and epics because they don't map cleanly to a single feature. Check every NFR against the epic breakdown specifically — is it referenced by at least the epics it should constrain? An NFR that appears in the requirements doc but nowhere in any epic is a real gap, not a technicality.

**Sequencing sanity.** If the epic breakdown includes a suggested delivery order or dependency chain, sanity-check it against the requirements: does an early epic depend on data or capability that a requirement implies won't exist until a later epic? This is a lighter check than the coverage ones — flag only clear, checkable contradictions, not judgment calls about prioritization.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: epics_review
schema_version: 1
initiative: <slug, from the two input files' frontmatter>
generated_by: epics-vs-requirements-checker
source_files: [<requirements.md filename>, <epics.md filename>]
date: <today, ISO format>
review_verdict: full_coverage | gaps_found | significant_gaps
---

# Traceability Check — [Initiative Name]: Requirements vs Epics

**Requirements source:** [doc/link]
**Epics source:** [doc/link]
**Checked:** [date]
**Overall result:** Full coverage confirmed | Gaps found | Significant gaps — do not proceed to build

## Summary

[3-5 sentences: coverage rate, the most important gaps if any, and a clear verdict on whether this epic breakdown is safe to hand to delivery as-is.]

## Coverage Matrix

| REQ-ID | Requirement (short) | Covered by | Status |
|---|---|---|---|
| REQ-BR-001 | ... | EPIC-001 | OK |
| REQ-FR-004 | ... | EPIC-002, EPIC-003 | Duplicate — pick one owner |
| REQ-NFR-002 | ... | — | **Gap** |

## Gaps (Requirements With No Epic)

| REQ-ID | Requirement | Severity | Note |
|---|---|---|---|
| REQ-NFR-002 | ... | High | No epic references this performance target anywhere |

Severity: **High** = a functional or business requirement, or an NFR that clearly constrains in-scope work, with zero coverage. **Medium** = partial/ambiguous coverage. **Low** = a requirement that's arguably out of scope for this program, worth a sanity-check question rather than urgent action.

## Orphan Epic Content (Scope With No Source Requirement)

| Epic | Content with no REQ-ID | Looks like | Recommendation |
|---|---|---|---|
| EPIC-002 | "Bulk export to CSV" | Unvetted scope addition | Confirm with stakeholder and backfill a requirement, or cut from epic |

## Duplicate Coverage

[Requirements claimed as primary by more than one epic, with a recommendation on which epic should own it]

## Sequencing Concerns

[Any dependency contradictions found — omit this section if none]

## Verdict

[One clear paragraph: is this ready to hand to the delivery team as-is, or does it need another pass? If gaps exist, name the specific fix needed before proceeding — don't leave it vague.]
```

Keep `review_verdict` in the frontmatter consistent with **Overall result** in the body. Save the file as `{slug}-epics-review.md` and tell the user the filename plus what to do with it: if `review_verdict` isn't `full_coverage`, hand the findings back to `requirements-to-epics` for a revision; if it is, `{slug}-epics.md` is clear to proceed to `epic-to-story-decomposer`.

## Calibration notes

Do the ID-by-ID matching exhaustively — this is a mechanical cross-reference check, and skipping requirements because they "obviously" must be covered somewhere is exactly how gaps slip through in real reviews. If the requirements document uses IDs and the epic breakdown doesn't reference them consistently, do the mapping yourself by matching requirement text to epic content rather than giving up and reporting "epics don't use REQ-IDs" as the whole finding — that's a real finding too, but it shouldn't stop you from still doing the underlying coverage check.

If either input document is missing entirely (e.g., the user only has epics with no source requirements to check against), say so and explain you can't do a traceability audit without both sides — offer to review the epics on their own merits instead, or suggest `requirements-writer` if requirements don't exist yet.
