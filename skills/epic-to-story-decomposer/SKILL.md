---
name: epic-to-story-decomposer
description: Breaks a single epic (or a full epic breakdown) down into sprint-sized user stories, each following the INVEST criteria with Gherkin-style Given/When/Then acceptance criteria and explicit traceability back to the epic and underlying requirement IDs. Use this whenever the user wants to "break an epic into stories," "write user stories for this," "get this ready for sprint planning/backlog grooming," or asks "what stories come out of this epic." Also trigger when the user hands over an epic description and wants it turned into backlog-ready tickets, even without using the word "story" explicitly — "what would we actually build first" or "how do we slice this epic" are the same request. This is the last step before work becomes plannable in a sprint; it assumes epics already exist (use requirements-to-epics first if they don't).
---

# Epic to Story Decomposer

## Pipeline contract

This skill is stage 7 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | nfr-elicitor | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | requirements-to-epics | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | epics-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | **epic-to-story-decomposer** (you are here) | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | story-checker | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is the kebab-case initiative identifier carried in every document's frontmatter — read it from the input file's `initiative` field and carry it forward unchanged.

**Input contract for this skill:** expects a file with `doc_type: epics` frontmatter, containing one epic or several. If a `{slug}-epics-review.md` exists and its `review_verdict` isn't `full_coverage`, flag that to the user before proceeding — decomposing an epic with known coverage gaps just carries the gap one level deeper into the backlog.

**Output contract for this skill:** produce one output file per epic you decompose, named `{slug}-epic-<EPIC-ID>-stories.md`, each with the frontmatter block shown in Output Format below. If the user asks you to decompose every epic in the input at once, write one file per epic rather than merging them — it keeps each file's `epic_id` unambiguous and keeps any one file from growing unwieldy — and list all the filenames you produced when you're done. These files feed `story-checker` (stage 8), the quality gate that validates the story slice against its epic before the backlog is considered sprint-ready.

## What a good story slice looks like

A story is a vertical slice of user- or system-observable value, not a horizontal slice of technical work. "Build the database schema" is not a story — nobody outside the team can see or verify it's done, and it delivers nothing on its own. "A customer can submit a claim with required documents attached" is a story — it's demoable, testable, and someone could, in principle, ship just this and stop.

The classic trap when decomposing epics is slicing by architecture layer (backend story, frontend story, API story) instead of by capability. Layer-sliced "stories" create false dependencies (frontend story is blocked until backend story is "done," but neither is independently valuable or testable) and hide the actual scope of what's being built until everything is finished. Slice vertically wherever possible: each story should touch whatever layers it needs to deliver one complete, observable behavior.

## Process

1. **Read the epic's full context first** — its rationale, its requirement list, its acceptance boundary (what's in vs. out). Pull the actual requirement text for every REQ-ID the epic references, not just the ID; the requirement's acceptance criteria (if it has them from `requirements-writer`) are often close to a first draft of story-level AC.

2. **Identify the distinct user-observable behaviors inside the epic.** Look for natural seams: different actors, different triggers/entry points, the happy path versus distinct edge cases that are big enough to warrant their own story (versus edge cases that belong inside one story's AC). A rule of thumb: if an edge case needs its own significant logic and would meaningfully change the size of the story if bundled in, it's probably its own story; if it's a one-line AC addition, keep it bundled.

3. **Write each story as: "As a [actor], I want [capability], so that [benefit]."** The "so that" clause isn't decoration — if you can't write a real benefit, it's a signal this isn't actually a coherent independent slice, or that you've drifted into describing a task rather than a need.

4. **Apply INVEST to every story before finalizing it:**
   - **Independent** — can this be built and shipped with minimal ordering dependency on other stories in the same epic? (Some ordering is unavoidable — e.g., you generally need to be able to create a thing before you can edit it — call these out explicitly rather than pretending independence.)
   - **Negotiable** — is this a statement of need, not an over-specified implementation plan the team has no room to figure out?
   - **Valuable** — does this deliver something observable to a user, an operator, or a downstream system — not just an internal technical milestone?
   - **Estimable** — is there enough detail that a team could size it without a big unresolved unknown? If not, flag it as needing a spike first rather than forcing a fake estimate-ready story.
   - **Small** — could a team plausibly finish this within a single sprint? If a story looks like it's actually epic-sized, split it further rather than shipping an oversized story.
   - **Testable** — same bar as requirements: could QA write a pass/fail test from the acceptance criteria alone?

5. **Write acceptance criteria in Given/When/Then form**, covering the happy path plus the significant edge/error cases that are this story's responsibility (not every edge case in the universe — just the ones a reasonable reviewer would expect this story to handle).

   **If the story delivers AI/ML/non-deterministic behavior** — because it traces to a requirement `requirements-writer` marked AI/ML, or because the behavior plainly won't return the exact same result twice even though nobody tagged it — don't force it into Given/When/Then. Carry the metric/threshold/enforcement shape down from the source requirement instead: name the metric, the threshold (inherit the requirement's if it applies at this story's scope, or state a story-specific one if the requirement's threshold is an aggregate the story only partially contributes to), and Gate/Monitor. A story can still carry ordinary Given/When/Then AC alongside this for its deterministic edges (e.g. "Given no results meet the confidence threshold, when the model runs, then the UI shows the fallback state") — the two AC styles aren't mutually exclusive within one story, they cover different behavior within it.

6. **Trace every story back to its epic and, where possible, the specific requirement(s) it fulfills.** ID stories `STORY-001`, `STORY-002` sequentially, scoped within the epic (or globally if the user is decomposing multiple epics at once — ask if it's ambiguous which numbering they want).

7. **Check full coverage before finishing**, the same discipline as the epic-breakdown skill: does every requirement the epic claims to cover show up in at least one story's scope? If a chunk of the epic doesn't cleanly decompose (e.g., it's really an NFR that constrains every story rather than being its own story), say so explicitly — don't force an NFR into a fake standalone story just to make the coverage table look complete.

## Output format

Produce a single Markdown **file per epic** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: stories
schema_version: 1
initiative: <slug, from the epics file's frontmatter>
generated_by: epic-to-story-decomposer
source_file: <the epics.md filename this was derived from>
epic_id: EPIC-XXX
date: <today, ISO format>
id_prefixes_used: [STORY]
---

# [Epic Name] (EPIC-XXX) — Story Breakdown

**Epic:** EPIC-XXX — [name]
**Date:** [date]
**Requirements covered by this epic:** [REQ-IDs]

## Summary

[2-3 sentences: how many stories, the overall slicing approach, and any notable sequencing or spike needs.]

## Stories

### STORY-001: [Short title]

**As a** [actor], **I want** [capability], **so that** [benefit].

**Traces to:** EPIC-XXX / REQ-FR-XXX
**Size (T-shirt or points, per team convention):** [S/M/L or leave placeholder for team estimate]
**Depends on:** [STORY-IDs, or "None"]

**Acceptance Criteria:**
- Given [context], when [action], then [outcome]
- Given [edge case], when [action], then [outcome]

**AI/ML Acceptance Criteria (if this story delivers non-deterministic behavior):**
- Metric: [...] · Threshold: [...] · Enforcement: Gate | Monitor

[Repeat per story]

## Sequencing Notes

[Genuine ordering dependencies between stories, if any — distinct from a suggested sprint plan, which is a team planning decision this skill shouldn't presume to make]

## Coverage Check

| Requirement/Epic scope item | Covered by story | Note |
|---|---|---|
| REQ-FR-002 | STORY-001, STORY-003 | |
| [NFR constraining whole epic] | — | Cross-cutting — applies to all stories, not a standalone story |

## Open Questions / Spike Candidates

[Anything too unclear to size or slice confidently — flag it rather than guessing]
```

Save the file as `{slug}-epic-<EPIC-ID>-stories.md` and tell the user the filename(s) plus what's next: `story-checker` to validate each story breakdown against its epic before it's treated as sprint-ready.

## Common failure modes to avoid

**Slicing by technical layer.** Covered above — this is the single most common mistake. If you notice you've produced a "backend story" and a "frontend story" for the same capability, merge them into one vertical story or re-slice by user scenario instead.

**Writing acceptance criteria that just restate the story title.** "Given the user wants to submit a claim, when they submit it, then it's submitted" adds nothing. Acceptance criteria should surface the specific rules, validations, and edge cases that make this story non-trivial.

**Over-slicing.** Ten one-line stories for what's really one small, cohesive piece of work creates backlog noise and loses the connective "so that" value story. If several candidate stories share the same actor, trigger, and benefit and differ only in minor detail, they're probably AC on one story, not three stories.

**Forcing NFRs into fake stories.** A performance or security constraint that applies across the whole epic isn't a story — it's a constraint on every story. Note it as cross-cutting rather than inventing a "make it fast" story with no independent value.

**Forcing an AI/ML story's behavior into Given/When/Then.** "Given the user asks a question, when the model responds, then the response is good" isn't checkable — it's a Given/When/Then costume on top of an inherently non-deterministic behavior. If the underlying behavior can't be pinned to an exact expected output, use the metric/threshold/enforcement AC shape instead, inherited from the source requirement.
