---
name: requirements-to-epics
description: Analyzes a set of business/functional/non-functional requirements (a REQ-ID table, a BRD, or a raw requirements list) and groups them into coherent epics — with rationale, suggested sequencing, and explicit traceability back to every source requirement ID. Use this whenever the user wants to "turn requirements into epics," "break this down for the backlog," "figure out how to sequence this work," or asks "what epics come out of this" after sharing a requirements document. Also trigger when the user has a big pile of requirements and wants to know how to organize delivery, even if they don't use the word "epic" — "how would we structure this as a program of work" or "what are the workstreams here" are the same request. This skill produces the epic breakdown; pairing it with epics-vs-requirements-checker afterward validates nothing was dropped.
---

# Requirements to Epics

## Pipeline contract

This skill is stage 5 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | nfr-elicitor | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | **requirements-to-epics** (you are here) | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | epics-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | epic-to-story-decomposer | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | story-checker | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is the kebab-case initiative identifier carried in every document's frontmatter — read it from the input file's `initiative` field and carry it forward unchanged into your own output.

**Input contract for this skill:** expects a file with `doc_type: requirements` frontmatter, and — if it exists — `{slug}-nfrs.md` too. When `{slug}-nfrs.md` is present, treat it as the authoritative NFR source (per `nfr-elicitor`'s contract) rather than the thin NFR table inside requirements.md, and reference its `REQ-NFR-*` IDs when noting which epics an NFR constrains. If `{slug}-nfrs.md` doesn't exist yet, fall back to the NFR table inside requirements.md and don't block on it — NFR elicitation can happen before or after epic breakdown, and re-running this skill later to backfill NFR references is normal. If you're handed something without `doc_type: requirements` frontmatter at all (a raw list, a doc from outside this pipeline), you can still do the analysis, but set `source_file: null` and say plainly in your Summary that this wasn't run against a pipeline-native requirements file, so coverage claims can't be cross-checked automatically by `epics-vs-requirements-checker` later. If a `{slug}-requirements-review.md` exists and its `review_verdict` isn't `ready`, flag that to the user before proceeding — epic breakdown built on unreviewed requirements may need rework once the review lands.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-epics.md`, with the frontmatter block shown in Output Format below. Tell the user the filename when done.

## What an epic actually is

An epic is a grouping of related requirements that delivers a coherent, independently valuable slice of the overall need — not just a folder for convenience. A good epic answers "why would we ship this together, and what can a user or the business do once it's done that they couldn't before?" A bad epic grouping is either too granular (an epic per requirement, which defeats the point of grouping at all) or too broad (one epic containing the entire project, which defeats the point of breaking it down).

The test for a well-formed epic: if you shipped only this epic and nothing else in the initiative, is there a real, describable capability the business now has? If the answer is "no, it's only useful combined with three other epics," reconsider the grouping or at least make the dependency explicit.

## Process

1. **Read every requirement before grouping anything.** Build a mental (or literal, in scratch notes) map of the requirement set: which requirements share an actor, a data entity, a business process step, or a system component. Grouping by these natural seams produces epics that make sense to both engineers and business stakeholders — grouping by superficial similarity (e.g., "all the ones that mention 'user'") usually doesn't.

2. **Group requirements into epics.** Every functional and business requirement must end up in exactly one primary epic (a requirement can be *referenced* by more than one epic if it's a genuine cross-cutting dependency, but it should have one home). Non-functional requirements are different — they often apply across multiple epics (e.g., a performance NFR might constrain five different epics' worth of endpoints). Don't force an NFR into a single epic if it's genuinely cross-cutting; note it against every epic it constrains, or create a cross-cutting "Platform/NFR" epic if there's a critical mass of them.

3. **Name each epic for the capability it delivers**, not for the mechanism. "Order cancellation and refunds" is a good epic name. "Refund microservice" describes an implementation choice that hasn't been decided yet — avoid baking architecture into epic names at this stage.

4. **Write a one-paragraph rationale per epic**: why these requirements belong together, and what becomes possible once the epic ships.

5. **Suggest a sequence**, not just a list. Identify genuine dependencies (epic B needs data or capability that only exists once epic A ships) versus soft preferences (this would just be nice to do first). Call out dependencies explicitly — a sequence with unstated dependencies is a sequence someone will get wrong in planning.

6. **Size epics roughly**, using relative terms (S/M/L/XL) based on the number and complexity of requirements inside, not fake story points — real sizing happens at the story level, which is a later skill's job. This is just enough signal for prioritization conversations.

7. **Give every epic a stable ID**: `EPIC-001`, `EPIC-002`, sequential. Don't renumber existing epics when adding new ones to an existing breakdown.

8. **Double check full coverage before finishing.** Every REQ-ID from the input should appear in at least one epic's requirement list. If you deliberately excluded a requirement (e.g., it's genuinely out of scope for this program of work), say so explicitly in an "Excluded" note rather than silently dropping it — silent drops are exactly what the paired checker skill exists to catch, so don't make it do the finding you could have surfaced yourself.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: epics
schema_version: 1
initiative: <slug, from the requirements file's frontmatter>
generated_by: requirements-to-epics
source_file: <the requirements.md filename this was derived from, or null>
date: <today, ISO format>
id_prefixes_used: [EPIC]
---

# [Initiative Name] — Epic Breakdown

**Derived from:** [source requirements doc/link]
**Date:** [date]
**Total requirements in source:** [N]
**Requirements covered by epics below:** [N] ([N] excluded — see Excluded Requirements)

## Summary

[2-4 sentences: how many epics, the overall shape of the breakdown (e.g., "organized around the claim lifecycle: intake, review, payout, and reporting"), and any major sequencing call-out.]

## Epics

### EPIC-001: [Name]

**Delivers:** [the capability this unlocks, in one sentence]
**Rationale:** [why these requirements are grouped together]
**Requirements covered:** REQ-BR-001, REQ-FR-002, REQ-FR-003, ...
**Relevant NFRs:** REQ-NFR-001 (if applicable — note, don't duplicate the full text)
**Size:** S | M | L | XL
**Depends on:** [EPIC-IDs, or "None"]

[Repeat per epic]

## Suggested Sequence

| Order | Epic | Why here | Blocking dependency |
|---|---|---|---|
| 1 | EPIC-001 | ... | None |
| 2 | EPIC-003 | ... | EPIC-001 (needs X) |

## Coverage Check

| REQ-ID | Covered by | Note |
|---|---|---|
| REQ-BR-001 | EPIC-001 | |
| REQ-FR-007 | — | Excluded — [reason] |

## Excluded Requirements

[Any requirement deliberately left out of every epic, with reasoning. Leave this section out entirely if coverage is complete — don't write "none" as filler, just omit the section.]

## Open Questions

[Anything where the grouping or sequencing depends on a business decision you couldn't infer from the source material — e.g., which of two plausible groupings the stakeholders would actually prefer.]
```

Save the file as `{slug}-epics.md` and tell the user the filename plus what's next: `epics-vs-requirements-checker` to validate coverage, or straight to `epic-to-story-decomposer` per epic.

## Common failure modes to avoid

**Grouping by document section instead of by capability.** If the source requirements doc already has section headers, don't just turn each section into an epic mechanically — check whether the document's organization actually reflects delivery-sized chunks, or whether it was organized for readability (e.g., "all the NFRs" as one section) in a way that doesn't map to epics at all.

**Losing NFRs in the shuffle.** It's easy to grope through functional requirements and forget the non-functional ones exist. Explicitly sweep the NFR list against your epic breakdown before finishing.

**Over-indexing on requirement count for sizing.** Five simple, similar requirements might be smaller than two requirements that each imply a new integration or data migration. Size on real complexity signals in the requirement text (new external system, new data entity, unclear/ambiguous acceptance criteria implying design work), not just a headcount of REQ-IDs.

**Silent scope narrowing.** If you genuinely can't figure out where a requirement belongs, that's a finding to surface as an open question — not a reason to quietly drop it from the coverage table.
