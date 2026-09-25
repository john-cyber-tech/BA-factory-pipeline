---
name: adr-writer
description: Writes Architecture Decision Records (ADRs) — a structured record of a real architectural choice, the genuine alternatives that were considered, the trade-offs, the decision, and its consequences (including downsides, not just upsides) — traced back to the requirements, NFRs, and epics that drove it. Trigger on "write an ADR," "architecture decision record," "document this decision," "record why we chose X," "what are our options for [technical choice]," "should we go with [option A] or [option B]," or whenever epics/NFRs clearly imply a technical decision point that hasn't been formally recorded yet (e.g. an aggressive availability NFR implies a decision about redundancy strategy). Use this any time a real technical fork-in-the-road needs a durable, honest record — not a rubber-stamp justification for a decision someone already made without considering alternatives.
---

# ADR Writer

## Pipeline contract

This skill is stage 9 of an eleven-stage business-analysis-to-architecture pipeline. Each stage reads and/or writes a plain Markdown file carrying a small YAML frontmatter block, and one stage's output file is fed directly as the next stage's input file.

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
| 9 | **adr-writer** (you are here) | `{slug}-epics.md` + `{slug}-nfrs.md` | `{slug}-adr-<ADR-ID>-<short-name>.md` (`doc_type: adr`) |
| 10 | solution-architecture-writer | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is a short kebab-case initiative identifier carried in every document's frontmatter and reused across the whole chain. Gate stages (3, 6, 8, 11) route their findings back to the stage that produced the document under review.

**Input contract for this skill:** read `{slug}-epics.md` and `{slug}-nfrs.md` before writing anything. The decision to document either comes directly from the user ("write an ADR for how we handle X"), or is surfaced by you scanning epics and NFRs for places where multiple viable technical approaches exist and no decision is on record yet. If `{slug}-nfrs.md` wasn't supplied, you can still write an ADR, but flag to the user that decision drivers may be incomplete without it, and suggest running `nfr-elicitor` first if the decision genuinely turns on quality attributes — which most architecture decisions do (latency, availability, consistency, cost ceilings, compliance).

**Output contract for this skill:** this skill runs multiple times per initiative — once per decision — so output is one file per decision, always written as an actual `.md` file, named `{slug}-adr-<ADR-ID>-<short-kebab-slug-of-the-decision>.md` (e.g. `claims-portal-adr-003-caching-strategy.md`). IDs are sequential per initiative (`ADR-001`, `ADR-002`, ...) and are never renumbered once assigned, even if an earlier ADR is later superseded — superseding creates a new ADR that references the old one, it does not overwrite it. When you're done, tell the user the filename and remind them that `solution-architecture-writer` will read this file and is expected to respect this decision when it runs.

## What makes an ADR worth writing

An ADR earns its place if it documents a genuine fork: a point where more than one approach was actually viable given the real constraints on this initiative. If, honestly, there was only one sane option — because of a hard compliance requirement, an existing platform commitment, a cost ceiling, whatever — say that plainly in the record rather than inventing a straw-man alternative just to make the document look like a thorough comparison. A one-option ADR that says so honestly is more useful than a three-option ADR where two of the options were never real.

The value of an ADR is not the decision itself — with hindsight, the right call often looks obvious. The value is the durable record of *why*: what was considered, what was rejected and on what grounds, what constraints were in force at the time. Six months from now, someone should be able to read the ADR and either agree with the reasoning or see exactly which assumption has since changed — without re-litigating the whole debate from scratch or accidentally reversing a decision whose original rationale they never saw.

## Process

1. **Identify the decision.** Either the user names it directly, or you find it by scanning `{slug}-epics.md` and `{slug}-nfrs.md` for signals that a technical choice is needed. A strict latency NFR paired with a high-throughput epic implies a caching or scaling decision. A strict availability NFR implies a redundancy or failover decision. A data-residency or compliance NFR implies a hosting or data-partitioning decision. Name the decision precisely — "how do we cache product data," not "performance."

2. **List genuinely viable options.** At least two, ideally three. Do not set up a weak option purely to make the chosen one look better by comparison. If you genuinely cannot find more than one viable option given the real constraints, say so explicitly in the Context or Decision section rather than padding the options table with a fake alternative.

3. **Evaluate every option against the actual constraints on record.** Cite specific `REQ-NFR-*` and `EPIC-*` IDs — argue from what's actually written in the source documents, not from abstract best practice. Build an honest trade-off table that shows what each option is good and bad at, not just a pro/con list stacked in favor of the eventual winner.

4. **State the decision plainly.** Name the chosen option and explain the deciding factor or factors in plain language — what tipped it, given the trade-offs just laid out.

5. **State consequences honestly.** What gets easier and what gets harder as a result of this choice. What future flexibility is given up. Any follow-on decisions this creates (a decision to use event sourcing, for example, creates a follow-on decision about replay/versioning strategy — name it even if you're not resolving it here).

6. **Assign a stable ID.** `ADR-XXX`, sequential per initiative, never renumbered. If this ADR changes a decision made in an earlier one, do not edit the original — write a new ADR, set `status: superseded` on the old file, and cross-reference both directions.

7. **Trace the decision back to its drivers.** Cite the specific `REQ-NFR-*`, `REQ-FR-*`, and `EPIC-*` IDs that actually motivated this decision. An ADR with no cited driver is a red flag that it's solving a problem nobody on record actually has — pause and ask the user what's really driving it before finishing the document.

8. **Write the Agent-Readable Summary last, after everything else is settled.** It must be self-contained — readable and actionable without the rest of the ADR — and must name the rejected alternative(s) explicitly, not just the chosen option. A summary that only states what was chosen ("Use Kafka") gives an agent no reason not to also reach for RabbitMQ next week; a summary that states what was chosen *and* rejected ("Use Kafka; do not introduce RabbitMQ or SQS") actually prevents the re-litigation this field exists to stop.

## Output format

Produce a single Markdown **file** — the frontmatter below plus the body template:

```markdown
---
doc_type: adr
schema_version: 1
initiative: <slug>
generated_by: adr-writer
decision_id: ADR-XXX
status: proposed | accepted | superseded
supersedes: <ADR-ID or null>
source_files: [<epics.md filename>, <nfrs.md filename>]
date: <today, ISO format>
---

# ADR-XXX: [Decision Title]

**Status:** proposed | accepted | superseded (by ADR-YYY)
**Date:** <ISO date>
**Deciders:** [if known; otherwise omit the line]

**Agent-Readable Summary:** [One or two imperative sentences a coding agent can parse and apply as a hard constraint without reading the rest of the file — name what to use, what NOT to use, and why in one clause. E.g. "Use Apache Kafka for all event streaming; do not introduce RabbitMQ, SQS, or another broker for this purpose — team already operates Kafka at scale and a second broker technology was explicitly rejected for operational overhead."]

## Context

[2-4 sentences: what forced this decision. What situation, requirement, or constraint made "do nothing" not an option.]

## Decision Drivers

- REQ-NFR-XXX: [what it demands and why it bears on this decision]
- EPIC-XXX: [what it demands and why it bears on this decision]
- [any other concrete, cited driver — no uncited drivers]

## Options Considered

| Option | Pros | Cons | Fit against key drivers |
|---|---|---|---|
| A: [name] | ... | ... | ... |
| B: [name] | ... | ... | ... |
| C: [name] | ... | ... | ... |

[If genuinely only one viable option exists, replace this table with a short paragraph stating that plainly and why.]

## Decision

We will use **[chosen option]**. [Plain-language explanation of the deciding factor(s) — why this option wins given the drivers above.]

## Consequences

### Positive
- [what gets easier or better]

### Negative
- [what gets harder, what flexibility is given up, what debt this incurs — be honest here]

## Related Decisions

- [Other ADRs this depends on, conflicts with, or supersedes — or "None yet on record."]
```

Save the file as `{slug}-adr-<ADR-ID>-<short-kebab-slug-of-the-decision>.md`. Confirm the filename to the user and note that `solution-architecture-writer` will read it, alongside any other ADRs on record, when it builds the architecture document in stage 10.

## Common failure modes to avoid

**Staging a fake comparison.** If asked to "document why we chose X" and only X was ever genuinely viable, say that plainly instead of inventing weak alternatives to lose a rigged contest — a rigged ADR is worse than no ADR, because it looks like due diligence that never happened.

**Writing consequences as marketing copy.** Every real decision costs something. If the Negative section is empty or reads like a sales pitch, you haven't looked hard enough — go back and find what this choice actually gives up.

**Documenting implementation detail instead of a decision.** An ADR isn't a design doc or a how-to guide. If there's no real alternative being rejected, it's not a decision — it's a detail, and it doesn't belong in this format.

**Citing no requirement, NFR, or epic driver.** An ADR with no traceable driver is a sign the decision is solving a problem nobody has actually raised. Chase down the real driver before finishing the document, or ask the user what's actually motivating it.

**Writing an Agent-Readable Summary that only states the choice, not the rejection.** "We use PostgreSQL" doesn't stop an agent from also standing up MongoDB for a new feature next quarter. The summary must name what NOT to do, as explicitly as what to do.
