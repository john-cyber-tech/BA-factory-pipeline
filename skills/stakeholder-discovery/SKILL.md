---
name: stakeholder-discovery
description: Turns raw stakeholder interview notes, workshop transcripts, kickoff meeting notes, or a loose list of "who wants what" into a structured stakeholder needs document — who the stakeholders/actors are, what each one needs and why, a RACI, and a set of discrete, solution-free "needs" statements each with a stable ID, ready to feed requirements-writer. Use this whenever the user wants "stakeholder discovery," to know "who are our stakeholders," to "capture these interview notes," to process "workshop notes" or "kickoff notes," to build "stakeholder needs" or a "RACI," or asks "who wants what here" after handing over raw meeting or interview material. This is the entry point of the whole BA pipeline — run it before requirements-writer whenever real discovery material (interview notes, transcripts, workshop output) exists to work from; skip straight to requirements-writer only when the user already has clear, pre-digested asks and no raw discovery material to process.
---

# Stakeholder Discovery

## Pipeline contract

This skill is stage 1 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file — no reformatting, no copy-pasting between tools, no losing traceability at a handoff.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | **stakeholder-discovery** (you are here) | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
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

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is a short kebab-case identifier for the initiative (e.g. `claims-portal-refresh`) that you choose here, at the very start of the chain, and that stays constant across every document every later stage produces — it's how a human or a script can tell which files belong together. Gate stages (3, 6, 8, 11) route their findings back to the stage that produced the document under review (2, 5, 7, 10 respectively) rather than flowing straight downstream — keep that in mind when you present results.

**Input contract for this skill:** you are the entry point of the pipeline, so there is no upstream file to require — your input is almost always freeform: interview notes, a workshop transcript, kickoff meeting minutes, an email thread, a pile of sticky notes someone typed up. That's expected; structuring it is the job. If instead you're handed a file with `doc_type: stakeholder_needs` frontmatter already, you're revising an existing document, not starting fresh: keep every existing NEED-ID stable (see step 4 below), and set `source_file` in your output frontmatter to that file's name so the revision history stays traceable. Pick `{slug}` from that file's `initiative` field if present; otherwise propose one from the initiative name and confirm it with the user, since every later stage inherits it.

**Output contract for this skill:** always write your result to an actual `.md` file (don't just print it in the chat) named `{slug}-stakeholder-needs.md`, with the frontmatter block shown in Output Format below. When you're done, tell the user the filename and that it's ready to hand to `requirements-writer`.

## What a need is, and why this stage exists before requirements

A need is the underlying problem, want, or goal a stakeholder actually has — not the system behavior that might eventually satisfy it. "Claims handlers need to know a claim's status without calling the claimant" is a need. "The system shall display claim status on a dashboard" is a requirement — a proposed solution to that need, and a different skill's job to write. Needs are captured before anyone starts designing a solution, and they stay solution- and technology-free on purpose: naming the fix too early forecloses better fixes nobody's thought of yet, and it hides the actual problem from everyone downstream who might want to solve it differently.

The single most common failure at this stage is skipping straight to "the system shall..." language, which quietly does requirements-writer's job badly and loses the "why" in the process — a requirement without a need behind it is much harder to evaluate, trade off, or defend later when priorities get squeezed. Your output here is deliberately upstream and softer-edged than a requirements document: it is allowed to contain open questions, thin spots, and named disagreements, because that's an honest picture of what discovery actually surfaced.

## Process

1. **Read all the raw material fully before extracting anything.** Don't start filling in stakeholders as you skim line by line. Read the whole transcript or note set first, then go back and identify every distinct person, role, or group mentioned — not just whoever spoke the most or was quoted directly. Notes often mention a stakeholder only once, in passing ("ops will hate this"), and that one mention is still a stakeholder to capture.

2. **For each stakeholder, capture role, needs, pain points, and influence/interest.** Role is who they are relative to the initiative (end user, approver, system owner, affected-but-silent party, etc.), not their job title alone. Pain points are what's wrong with the current state in their own words where possible. Influence/interest is a rough read on how much say they have and how much they care — useful for prioritizing whose needs get weight later, but it's a read, not a fact, so hedge it in the notes when the material is thin.

3. **Build a RACI for the initiative wherever the material actually supports it.** Who's Responsible, Accountable, Consulted, and Informed for the key decisions and activities the notes describe. If the material doesn't clearly establish ownership — which is common in early discovery — don't invent an owner to make the table look complete. Leave the cell blank and raise it explicitly in Open Questions instead; a fabricated RACI is worse than an honest gap because it will be trusted and acted on.

4. **Extract each discrete need as its own item, solution-free.** One need, one line: what's wanted, tagged with which stakeholder(s) raised it, and why it matters to them. Give each a stable ID — `NEED-001`, `NEED-002`, and so on — since these IDs are what requirements-writer will trace its own requirements back to. When revising an existing stakeholder-needs document, never renumber existing IDs; only append new ones.

5. **Actively look for needs that conflict, and name the conflict rather than resolving it.** Discovery material regularly contains stakeholders wanting incompatible things — compliance wants more approval steps, operations wants fewer; finance wants a hard cutoff date, engineering says it's not feasible. Don't silently pick a side, and don't blend two conflicting needs into one mushy compromise statement that satisfies neither. Surface the conflict plainly so it gets resolved by the people who actually own that decision.

6. **Don't let quiet or low-power stakeholders disappear.** If someone is clearly affected by the initiative but the material barely mentions them — end customers, a downstream support team, a compliance function nobody interviewed — that's a discovery gap, not a reason to leave them out. Name them, note what's missing, and flag it as an open question rather than either fabricating their needs or omitting them entirely.

## Output format

Produce a single Markdown **file** — the frontmatter below plus a Confluence-style body (headers, tables, no nested bullets more than one level deep — this renders cleanly whether it's opened as a file, pasted into Confluence, or read back in by the next skill). Use exactly this structure, frontmatter included:

```markdown
---
doc_type: stakeholder_needs
schema_version: 1
initiative: <slug>
generated_by: stakeholder-discovery
source_file: <input filename if you revised an existing stakeholder-needs.md, else null>
date: <today, ISO format>
id_prefixes_used: [NEED]
---

# [Initiative Name] — Stakeholder Needs

## Summary

[2-4 sentences: what this initiative is, what discovery material this was drawn from, and what state discovery is in — early/rough vs. fairly complete.]

## Stakeholders

| Name/Role | Goals | Pain Points | Influence/Interest |
|---|---|---|---|
| ... | ... | ... | ... |

## RACI

| Activity/Decision | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

## Needs

| NEED-ID | Need | Raised By | Rationale / Why It Matters |
|---|---|---|---|
| NEED-001 | ... | ... | ... |

## Conflicting or Competing Needs

[Each conflict named explicitly: which needs, which stakeholders, what the actual tension is. Don't resolve it here — that's a decision for the people who own it.]

## Open Questions

| # | Question | Why it matters | Owner |
|---|---|---|---|
```

The RACI table in particular is expected to be thin or partially blank when the material doesn't support it — leave cells empty and raise the gap in Open Questions rather than fabricating ownership to make the table look finished. Omit a section only if it's genuinely empty after a real attempt to populate it. Never omit the frontmatter block, even when the rest of the document is short — it's what makes this file machine-readable by the next skill in the chain.

Save the file as `{slug}-stakeholder-needs.md` and tell the user the filename plus that it's ready to hand to `requirements-writer`.

## Common failure modes to avoid

**Writing needs as requirements.** "The system shall send an email notification" is a requirement, not a need — it bakes in a solution before anyone's evaluated alternatives. The need underneath it is something like "the requester needs to know when their request is actioned." Leave the "shall" statements to requirements-writer; this skill's job ends at the need.

**Fabricating a RACI or influence/interest ratings the material doesn't support.** It's tempting to fill every cell so the document looks thorough, but an invented owner or a guessed influence level gets trusted downstream as fact. If the notes don't say who's accountable, say so in Open Questions instead of guessing.

**Letting one dominant voice drown out others actually mentioned in the material.** Interview notes and transcripts are rarely balanced — whoever talked the most or was interviewed longest tends to dominate the raw text. Deliberately check the material a second time for anyone mentioned only briefly or referred to in the third person, and give them their own line.

**Treating every stated want as equally valid and blending away real disagreement.** If two stakeholders want incompatible things, that's useful information, not noise to smooth over. Record both needs distinctly, tag who raised each, and call out the conflict explicitly rather than merging them into a compromise nobody actually asked for.
