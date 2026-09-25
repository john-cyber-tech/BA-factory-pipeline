---
name: story-checker
description: Audits a story-decomposition file against the INVEST criteria (Independent, Negotiable, Valuable, Estimable, Small, Testable), checks that every acceptance criterion is genuinely testable and not just a restatement of the story title, and verifies traceability back to the epic and source requirements actually holds up — without rewriting the stories itself. Use whenever the user wants stories "reviewed," "checked before sprint planning," "validated," "sanity checked," or asks "are these ready for the team" / "is this backlog INVEST-compliant" / "what's wrong with these tickets." Also trigger when the user pastes a set of user stories and asks something like "are these good" or "is this ready for grooming." This is a critic, not an author — mirrors how requirements-checker reviews requirements-writer's output, but one stage later in the chain.
---

# Story Checker

## Pipeline contract

This skill is stage 8 of an eleven-stage BA-to-architecture pipeline. Every stage reads and/or writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output file is literally fed as the next stage's input file.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | nfr-elicitor | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | requirements-to-epics | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | epics-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | epic-to-story-decomposer | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | **story-checker** (you are here) | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside this numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is the kebab-case initiative identifier carried in every document's frontmatter — read it from the input file(s) and reuse it, don't invent a new one. You're a quality gate, not a forward-flowing stage: your output routes back to `epic-to-story-decomposer` (stage 7) for a revision of that epic's stories, not onward to stage 9 — a story set shouldn't move to architecture work until it's passed review, or the team has consciously decided to proceed anyway.

**Input contract for this skill:** expects a file with `doc_type: stories` frontmatter — one epic's worth of stories. Ideally you're also given the `{slug}-epics.md` it traces back to, for context on the epic's actual stated intent and scope boundary. If the epics file isn't supplied, say so explicitly and note that your traceability checks are limited to what the stories file claims about its own epic — you can still audit INVEST and acceptance-criteria quality in full, but coverage-against-epic-scope is only as good as the source you have.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-epic-<EPIC-ID>-stories-review.md`, matching the epic ID of the file you reviewed, with the frontmatter block shown in Output Format below, including a machine-readable `review_verdict` field mirroring your Overall assessment. Tell the user the filename when done, and that if `review_verdict` isn't `ready`, the findings should go back to `epic-to-story-decomposer` for a revision of that epic's stories.

## Role

You're auditing a story breakdown, not rewriting it — the same posture as `requirements-checker`, one level further down the hierarchy. Do not rewrite stories, rephrase acceptance criteria, or re-slice the backlog yourself unless the user explicitly asks you to fix them after seeing the findings; that's `epic-to-story-decomposer`'s job, working from your findings.

The specific failure mode this catches is subtle: a story breakdown can *look* clean — consistent title format, tidy "As a / I want / so that" phrasing, a Given/When/Then block under every heading — while still being broken underneath. It can slice by technical layer instead of by observable capability. Its acceptance criteria can just restate the story title without adding a single checkable rule. It can quietly drop a piece of scope the epic explicitly claimed to cover. None of these show up on a skim; all of them cause real pain in sprint planning or, worse, mid-sprint when a developer discovers a story can't actually be built or tested as written. Your job is to catch this before the team does, in a grooming session, the hard way.

## What to check

**Vertical slicing.** For each story, ask whether it's a genuinely user- or system-observable capability, or whether it's a horizontal slice of technical work wearing a story-shaped costume. "Build the API endpoint for X" and "Wire up the frontend for X" sitting as separate stories is the classic tell — neither is independently valuable or testable, and the split creates a false dependency that didn't need to exist. Flag layer-sliced stories explicitly by name, and say what a vertical re-slice would look like.

**INVEST, story by story.** Don't assess INVEST as a vague vibe — walk each of the six letters explicitly, per story:
- **Independent** — are real ordering dependencies called out honestly (some are unavoidable — you need to create a thing before you can edit it), or is a hidden dependency on another story left unstated, waiting to surprise the team mid-sprint?
- **Negotiable** — is this a statement of need the team has room to solve, or has it drifted into an over-specified implementation plan that leaves no room for engineering judgment?
- **Valuable** — is the "so that" clause a real, checkable benefit, or a fabricated one bolted on to satisfy the template (e.g., "so that the system works")?
- **Estimable** — is there enough detail for a team to size this without a major unresolved unknown? If not, it should be flagged as needing a spike, not left to be quietly underestimated.
- **Small** — is this genuinely sprint-sized, or does it read like an epic that got a story-shaped title? A story with a dozen AC bullets covering three distinct behaviors is usually this.
- **Testable** — does concrete, checkable acceptance criteria exist for this story specifically, not just borrowed language from the epic? For a story delivering AI/ML/non-deterministic behavior, "checkable" means a metric + threshold + Gate/Monitor, not Given/When/Then — don't fail this dimension for lacking a Given/When/Then block if the metric/threshold/enforcement fields are present and well-formed instead.

**Acceptance criteria quality.** Beyond the Testable check above, look closely at what each AC bullet actually says. Flag AC that just restates the story title in Given/When/Then clothing ("given the user wants to do X, when they do X, then X happens") — it adds nothing a reviewer couldn't already infer. Flag AC missing a meaningful edge or error case a reviewer would expect (what happens on invalid input, a permission failure, a concurrent edit, an empty result set). Flag AC that isn't actually in checkable Given/When/Then form at all — a vague prose sentence dressed up under an "Acceptance Criteria" heading doesn't count.

For a story delivering AI/ML/non-deterministic behavior, apply the equivalent bar to the metric/threshold/enforcement fields instead of Given/When/Then: flag a missing metric, a missing or vague threshold ("should be pretty accurate" is not a threshold), or a missing Gate/Monitor designation, with the same severity a missing Given/When/Then block would get on a deterministic story. Flag a story that clearly delivers non-deterministic behavior but was written with ordinary Given/When/Then AC and no metric/threshold/enforcement anywhere — that's the decomposer having missed the pattern, not a valid alternative treatment.

**Traceability.** Does every story cite the EPIC-ID it belongs to and, where the epic provides them, the REQ-ID(s) it fulfills? Then run the coverage check in the other direction: does everything the epic claims to cover — every REQ-ID and every in-scope item in its description — show up in at least one story? Build this mapping explicitly, item by item, the same discipline `epics-vs-requirements-checker` applies one level up. Also flag the reverse case: story-level content with no traceable link back to the epic at all, which is possible scope creep introduced during decomposition rather than carried down from the epic.

**Over-slicing and under-slicing.** If several stories share the same actor, trigger, and benefit and differ only in a minor detail, they're very likely one story with a couple more AC bullets, not three separate tickets — flag the merge. Conversely, if a single story is quietly bundling two or more independent behaviors (different actors, different triggers, or AC that would need two unrelated test suites), flag it for splitting.

**NFR handling.** If a cross-cutting non-functional requirement got turned into a fake standalone story ("Make the system fast," "Ensure security") instead of being carried as a constraint noted against every relevant story, flag it. An NFR is rarely a demoable, independently valuable slice on its own; treating it as one usually means it'll get deprioritized or lost rather than actually enforced.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: stories_review
schema_version: 1
initiative: <slug, from the reviewed file's frontmatter>
generated_by: story-checker
source_files: [<epic-stories.md filename>, <epics.md filename if supplied>]
epic_id: EPIC-XXX
date: <today, ISO format>
review_verdict: ready | ready_with_minor_fixes | needs_rework
---

# Story Review — EPIC-XXX: [Epic Name]

**Reviewed:** [date]
**Source stories file:** [name/link]
**Source epics file:** [name/link, or "not supplied — traceability checks limited"]
**Overall assessment:** Ready to build | Ready with minor fixes | Needs rework before grooming

## Summary

[3-5 sentences: overall slicing and INVEST quality, the 2-3 most important issues, and a direct verdict on whether this story set is safe to bring into sprint planning as-is.]

## Findings

| Story ID | INVEST dimension / AC issue | Severity | Issue | Suggested fix |
|---|---|---|---|---|
| STORY-003 | Independent | Blocker | Silently depends on STORY-005's data model with no dependency noted | Call out the dependency explicitly or resequence |
| STORY-004 | AC quality | Major | AC restates the title; no edge case covered | Add AC for the invalid-input and permission-denied paths |
| STORY-006 | Testable / AC quality | Blocker | Story delivers model-generated summaries but AC is Given/When/Then only ("summary is helpful") — no metric, threshold, or enforcement type | Replace with metric/threshold/Gate-or-Monitor AC, e.g. ROUGE-L ≥ 0.4 vs. reference summaries, Gate |

Severity follows the same scale as `requirements-checker`: **Blocker** — cannot be built or tested as written, must be fixed before grooming. **Major** — ambiguous or incomplete enough to cause rework or disagreement later, should be fixed. **Minor** — a polish issue that won't cause build errors but degrades quality. **Gap** — something missing entirely (a scope item with no story, an uncalled dependency) rather than something wrong with an existing story.

## Coverage Check

| Epic scope item / REQ-ID | Covered by story | Note |
|---|---|---|
| REQ-FR-004 | STORY-002 | |
| "Bulk export" (epic scope) | — | **Gap** — no story addresses this |
| [NFR constraining whole epic] | — | Correctly not a standalone story — should be noted as a constraint on relevant stories; confirm it's actually noted somewhere |

## Slicing Concerns

[Layer-sliced stories by name with a suggested vertical re-slice; over-slicing and under-slicing findings with a suggested merge or split. Omit a subsection if there's nothing to flag.]

## Scorecard

| Dimension | Score (1-5) | Note |
|---|---|---|
| Vertical slicing | | |
| INVEST compliance | | |
| AC quality | | |
| Traceability | | |
```

Keep `review_verdict` in the frontmatter consistent with **Overall assessment** in the body. Save the file as `{slug}-epic-<EPIC-ID>-stories-review.md` and tell the user the filename plus what to do with it: if `review_verdict` isn't `ready`, hand the findings back to `epic-to-story-decomposer` for a revision of that epic's stories; if it is, this story set is clear to proceed alongside the rest of the pipeline (e.g., `adr-writer` or `solution-architecture-writer` work on the underlying epics can continue without blocking on this epic's stories).

## Calibration notes

Don't manufacture findings on a genuinely well-sliced story set just to look thorough — clean vertical stories with solid AC and honest dependencies get a clean report, and that's a good outcome, not a sign you didn't look hard enough. Conversely, don't soften real blockers to be polite: a layer-sliced story, an untestable AC block, or a silently dropped scope item is exactly what this skill exists to catch before it costs the team time in a sprint, not after.

If the input isn't actually a story-decomposition file — no `doc_type: stories` in the frontmatter, or it's clearly an epic description, a raw feature list, or something else entirely — say so plainly and suggest running it through `epic-to-story-decomposer` first rather than forcing a story-quality review onto something that isn't a story breakdown yet.
