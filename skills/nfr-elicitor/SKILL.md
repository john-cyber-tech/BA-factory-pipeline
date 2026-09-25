---
name: nfr-elicitor
description: Runs a systematic, checklist-driven elicitation pass over non-functional requirements — performance/latency, availability, scalability, security, data privacy/compliance, accessibility, browser/device support, auditability/logging, observability, error handling/recovery, data retention, cost — turning whatever the requirements doc and stakeholder material implies into concrete, testable NFR targets, and explicitly flagging categories with no identified constraint instead of silently skipping them. Trigger on "elicit NFRs," "non-functional requirements," "quality attributes," "performance requirements," "security requirements," "SLAs/constraints," "flesh out the NFRs," or whenever a requirements document exists but its non-functional coverage looks thin (a handful of NFRs or none at all). This is the dedicated deep-dive the lightweight NFR table inside requirements-writer's output was never meant to substitute for — run it any time NFRs matter enough to get architecture wrong if missed.
---

# NFR Elicitor

## Pipeline contract

This skill is stage 4 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file — no reformatting, no copy-pasting between tools, no losing traceability at a handoff.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | requirements-writer | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
| 3 | requirements-checker | `{slug}-requirements.md` | `{slug}-requirements-review.md` (`doc_type: requirements_review`) |
| 4 | **nfr-elicitor** (you are here) | `{slug}-requirements.md` (+ `{slug}-stakeholder-needs.md` if present) | `{slug}-nfrs.md` (`doc_type: nfrs`) |
| 5 | requirements-to-epics | `{slug}-requirements.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics.md` (`doc_type: epics`) |
| 6 | epics-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-epics.md` (+ `{slug}-nfrs.md` if present) | `{slug}-epics-review.md` (`doc_type: epics_review`) |
| 7 | epic-to-story-decomposer | `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories.md` (`doc_type: stories`) |
| 8 | story-checker | `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-epics.md` | `{slug}-epic-<EPIC-ID>-stories-review.md` (`doc_type: stories_review`) |
| 9 | adr-writer | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`). `{slug}` is a short kebab-case initiative identifier carried in every document's frontmatter and reused across the whole chain. Gate stages (3, 6, 8, 11) route their findings back to the stage that produced the document under review (2, 5, 7, 10 respectively) rather than flowing straight downstream.

**Input contract for this skill:** your primary input is `{slug}-requirements.md` (`doc_type: requirements`), specifically its thin Non-Functional Requirements table — that's your starting set, not your finished product. If `{slug}-stakeholder-needs.md` is available, read it too; stakeholders often mention constraints ("has to work on the factory floor tablets," "audit needs three years of records") that never made it into the formal requirements doc as an NFR line item. If you're instead handed a file with `doc_type: nfrs` frontmatter already, you're revising an existing catalog: keep every existing `REQ-NFR-*` ID stable and treat this as a continuation pass, not a rewrite. Pick `{slug}` from the input file's `initiative` field if present.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-nfrs.md`, with the frontmatter shown in Output Format below. Once this file exists, it is the **authoritative** NFR source for every downstream skill — `requirements-to-epics`, `epics-vs-requirements-checker`, `adr-writer`, `solution-architecture-writer`, and `architecture-vs-requirements-checker` should all prefer this file's NFR catalog over the thin table still sitting inside `{slug}-requirements.md`. Tell the user this explicitly when you finish: the two documents will now disagree in scope, and this one wins.

## Why NFRs get missed

Stakeholders describe what the system should *do*. They rarely volunteer how well, how fast, how safely, or how it should fail — not because they don't care, but because those constraints are invisible until something breaks in production, and nobody thinks to narrate the absence of a problem. Asking "any non-functional requirements?" during elicitation reliably produces "no" or silence, not because there are none, but because the question is too open-ended to prompt recall.

The fix is procedural, not conversational: walk a fixed checklist of NFR categories and force an explicit answer for each one, even when that answer is "no constraint identified, needs confirmation." A blank is a silent gap that architecture will later fill in by accident. An explicit "no constraint identified" is a visible gap that someone can still choose to close. That distinction is the entire value this skill adds over the thin table requirements-writer already produced.

## Process

1. **Pull forward existing NFRs.** Extract every `REQ-NFR-*` entry already in `{slug}-requirements.md`, keep their IDs and wording, and treat them as your starting rows — don't rewrite or renumber them, and don't create duplicate entries for the same constraint under a new ID. Your numbering continues from the highest existing `REQ-NFR-XXX`, it doesn't restart.

2. **Work the fixed checklist, category by category.** For each of the thirteen categories below, decide whether the requirements doc, the stakeholder needs doc, or direct reasoning about the domain supports a concrete target. Write one of two things — never leave a category out of the table entirely:
   - a specific, testable target with a number, threshold, or defined condition attached, or
   - "No constraint identified — confirm with stakeholder," paired with a matching entry in Open Questions.

   Categories: Performance & Latency, Throughput & Capacity, Availability & Uptime/DR, Scalability, Security & Access Control, Data Privacy & Regulatory Compliance, Accessibility, Browser/Device/Platform Support, Auditability & Logging, Observability & Monitoring, Error Handling & Recovery, Data Retention & Archival, **Cost & Resource Ceilings**.

   Cost & Resource Ceilings covers anything with a per-unit or aggregate cost that can blow a budget silently: LLM/inference token spend per request or per user, third-party API call costs, compute/storage spend at scale, and any ceiling that should trigger a model-downgrade or throttling decision before it triggers a budget overrun. This category exists specifically because AI-touching features have cost profiles that scale with usage in ways traditional CRUD features don't — "no constraint identified" is a much riskier answer here than for most other categories, so push harder before accepting it.

3. **Attach a verification method to every real target.** For each NFR that does carry a concrete number or condition, add a short note on how someone would check it's actually met — a load test, a synthetic uptime probe, an access-control audit, a WCAG scan, a retention-policy report. An NFR with no way to verify it is exactly as useless as a functional requirement with no acceptance criteria, and gets discovered as useless at the same bad time: after it's built.

4. **Actively hunt for conflicts between NFRs.** A sub-200ms latency target and a mandatory synchronous full-audit-log write on every request can't both hold without a real architectural trade-off. A strict data-retention-minimization policy and a "keep everything for compliance" instruction from a different stakeholder are in direct tension. When you find this kind of collision, don't quietly resolve it by picking one side — record it in Conflicts / Tensions so a human makes the call.

5. **Cross-reference epics if they exist.** If the user also supplies `{slug}-epics.md`, note which epic(s) each NFR is likely to constrain in the Relevant Epics column. If no epics file exists yet, leave that column blank — `requirements-to-epics` is expected to fill it in when it runs, using this file as an input.

6. **Never fabricate a specific number.** Don't write "99.9% uptime" or "sub-second response" because it sounds like a plausible default for the domain — write it only if something in the source material, a regulation you can name, or an explicit stakeholder statement supports it. An invented number is worse than an honest gap: a gap gets asked about, a fabricated number gets built to a target nobody actually asked for and nobody notices was wrong until it's expensive to change.

## Output format

Produce a single Markdown **file** — the frontmatter below plus a Confluence-style body. Use exactly this structure, frontmatter included:

```markdown
---
doc_type: nfrs
schema_version: 1
initiative: <slug>
generated_by: nfr-elicitor
source_file: <the requirements.md filename this was derived from>
date: <today, ISO format>
id_prefixes_used: [REQ-NFR]
---

# [Initiative Name] — Non-Functional Requirements

## Summary

[2-4 sentences: how many NFRs were carried forward vs. newly elicited, how many categories landed a real target vs. "no constraint identified," and any conflict worth flagging up front.]

## NFR Catalog

| ID | Category | Requirement | Target/Metric | Verification Method | Relevant Epics | Source |
|---|---|---|---|---|---|---|
| REQ-NFR-001 | Performance & Latency | ... | ... | ... | ... | ... |

## Category Coverage Checklist

| Category | Addressed? | Note |
|---|---|---|
| Performance & Latency | Yes / No constraint identified | ... |
| Throughput & Capacity | Yes / No constraint identified | ... |
| Availability & Uptime/DR | Yes / No constraint identified | ... |
| Scalability | Yes / No constraint identified | ... |
| Security & Access Control | Yes / No constraint identified | ... |
| Data Privacy & Regulatory Compliance | Yes / No constraint identified | ... |
| Accessibility | Yes / No constraint identified | ... |
| Browser/Device/Platform Support | Yes / No constraint identified | ... |
| Auditability & Logging | Yes / No constraint identified | ... |
| Observability & Monitoring | Yes / No constraint identified | ... |
| Error Handling & Recovery | Yes / No constraint identified | ... |
| Data Retention & Archival | Yes / No constraint identified | ... |
| Cost & Resource Ceilings | Yes / No constraint identified | ... |

## Conflicts / Tensions

[Pairs or groups of NFRs that pull against each other, with the trade-off stated plainly. Omit only if a genuine review found none.]

## Open Questions

| # | Question | Why it matters | Owner |
|---|---|---|---|
```

Every one of the thirteen categories must appear as a row in the Category Coverage Checklist, no exceptions — that table is the whole point of running this skill instead of relying on the thin table in requirements.md. Never omit the frontmatter block; it's what makes this file machine-readable by every downstream skill.

Save the file as `{slug}-nfrs.md` and tell the user the filename plus that it supersedes the thin NFR table in `{slug}-requirements.md` for every downstream skill.

## Common failure modes to avoid

**Inventing precise numbers with no basis in the source material.** A confident-sounding "99.9% uptime" or "p95 under 300ms" that nothing in the requirements or stakeholder docs actually supports is worse than an honest gap — write "No constraint identified — confirm with stakeholder" instead, and let a human supply the real number.

**Treating "no constraint identified" as "doesn't matter."** It means the opposite: a business decision is still owed here. Every such row must carry a matching Open Question, not just a shrug in the table.

**Silently duplicating or renumbering the existing REQ-NFR entries from requirements.md.** This skill extends the existing catalog, it doesn't replace it wholesale — keep prior IDs and wording intact, and continue numbering from where they left off.

**Skipping a checklist category because the input never mentions it.** That silence is exactly what the checklist exists to surface. An absent category row is a missed elicitation pass, not a sign that the category doesn't apply — decide "doesn't apply here" explicitly, in writing, rather than by omission.
