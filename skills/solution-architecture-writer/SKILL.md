---
name: solution-architecture-writer
description: Produces a high-level solution architecture / HLD document from the epics, NFRs, and any accepted architecture decision records — a component breakdown (responsibility, interfaces, data owned), an architecture diagram (as an embedded Mermaid diagram inside the markdown file), explicit NFR-to-component mapping, and integration/data-flow description. Trigger on "design the architecture," "write an HLD," "high-level design," "what's the architecture for this," "solution design," "system design," "draw the architecture," or whenever epics and NFRs exist and the user wants a technical design that satisfies them. This is HIGH-LEVEL design — component responsibilities and integration points, not detailed schemas, class diagrams, or full API specs.
---

# Solution Architecture Writer

## Pipeline contract

This skill is stage 10 of an eleven-stage BA-to-architecture pipeline. Every stage reads and/or writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file — no reformatting, no copy-pasting between tools, no losing traceability at a handoff.

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
| 10 | **solution-architecture-writer** (you are here) | `{slug}-epics.md` + `{slug}-nfrs.md` + any `{slug}-adr-*.md` | `{slug}-architecture.md` (`doc_type: architecture`) |
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is a short kebab-case identifier for the initiative that stays constant across every document in the chain — it's how a human or a script can tell which files belong together.

**Input contract for this skill:** read `{slug}-epics.md` (what capabilities must be delivered) and `{slug}-nfrs.md` (the constraints the design must satisfy) — both are required for a design that's actually worth checking later. If `{slug}-nfrs.md` doesn't exist, still produce the architecture, but flag prominently, in both your response and the document's Open Questions section, that this design can't be validated against non-functional constraints that were never elicited, and suggest running `nfr-elicitor` first. Also read any `{slug}-adr-*.md` files present in the working set. These are decisions **already made**, not suggestions — treat every ADR with `status: accepted` as binding on this design. If the architecture you'd otherwise produce seems to want to contradict an accepted ADR, don't silently override it: flag the conflict explicitly (see Common failure modes below) and either conform the design to the ADR or surface the conflict as an open question for the user to resolve.

**Output contract for this skill:** always write your result to an actual `.md` file (don't just print it in the chat) named `{slug}-architecture.md`, with the frontmatter block shown in Output Format below. When you're done, tell the user the filename and that it's ready for `architecture-vs-requirements-checker` to validate against the original requirements and NFRs.

## What makes a component well-formed at this altitude

This skill operates at high-level design (HLD), not detailed design — the output is meant to orient a reviewer and a later, more detailed design effort, not to replace either. A component earns its place in the document if it satisfies three things:

It's named and scoped around a **business responsibility**, not a technology choice. "Claims Intake Service" tells a reader what the component is for; "the microservices layer" or "the API tier" doesn't — it's an implementation category standing in for a responsibility nobody named. If you can't state a component's purpose without referring to the tech stack, you haven't found the responsibility yet.

It has a **clear boundary** — what it owns (data, decisions, behavior) and, just as importantly, what it explicitly does not own. Overlapping or fuzzy boundaries between components are the most common source of downstream confusion and duplicated effort.

It's **traceable**. Every component's existence should map back to specific EPIC-IDs or REQ-IDs it serves. A component with no requirement driving it is unexplained scope — exactly what `architecture-vs-requirements-checker` will go looking for in stage 11, so don't make it find something you could have caught yourself.

## Process

1. **Read everything before designing anything.** Read `{slug}-epics.md` fully to understand what capabilities must exist end to end, read `{slug}-nfrs.md` for the constraints the design must satisfy (throughput, latency, availability, security, compliance), and read every `{slug}-adr-*.md` file for decisions already locked in. Treat `status: accepted` ADRs as binding; treat `proposed` or `rejected` ADRs as context, not constraints.

2. **Identify the major components/services needed to deliver the epics**, respecting any accepted ADRs (e.g., if an ADR already commits to an event-driven integration style or a specific data store, design around that rather than re-deciding it). Name each component for the responsibility it holds, not the technology it happens to run on.

3. **For each component, define four things**: what it's responsible for, its key interfaces/contracts at a summary level (the shape of what it exposes or consumes — not a full API spec, that's a later design phase), what data it owns, and which EPIC-IDs/REQ-IDs it satisfies. If a component can't be traced to at least one epic or requirement, either find the traceability or cut the component.

4. **Describe integration points and data flow between components, and draw it.** Include one Mermaid diagram (a flowchart or component-style graph, in a fenced ```mermaid code block) showing the components and their connections. A design that's only prose is much harder for a reviewer or a later engineer to actually use than one with a picture — don't skip the diagram to save time.

5. **Map every NFR from `{slug}-nfrs.md` to the component(s) responsible for satisfying it**, with a short note on *how* — e.g. "REQ-NFR-004 (sub-200ms p95 read latency) — satisfied by Claims Cache Service via read-through caching." An NFR with no component mapped to it is a gap, not something to leave implicit for the reviewer to spot; find it yourself and either assign it or flag it as unaddressed.

6. **Surface anything that doesn't map cleanly.** If an epic or an NFR doesn't fit naturally onto any component you've designed, say so explicitly in Open Questions/Risks rather than forcing a bad fit or quietly leaving it out.

7. **Give every component a stable ID** (`COMP-001`, `COMP-002`, ...), assigned sequentially. Never renumber existing IDs on a later revision of this document — later skills and any traceability matrix key off them.

8. **Run a pre-mortem before finishing.** Assume this design has already shipped and failed badly in production six months from now — not "might fail," *did* fail. Write down the two or three most plausible reasons why, working backward from the failure rather than forward from the design. Typical angles: a component boundary that looked clean on paper but forces two teams to coordinate every release; an NFR mapping that works at expected load but not at 5x load; a single component silently becoming a bottleneck or a single point of failure; an integration assumption about an external system that turns out to be wrong. This is a distinct, adversarial pass — don't just restate items already sitting in Open Questions/Risks.

## Output format

Produce a single Markdown **file** — the frontmatter below plus a Confluence-style body:

```markdown
---
doc_type: architecture
schema_version: 1
initiative: <slug>
generated_by: solution-architecture-writer
source_files: [<epics.md filename>, <nfrs.md filename>, <any adr filenames used>]
date: <today, ISO format>
id_prefixes_used: [COMP]
---

# [Initiative Name] — Solution Architecture

**Status:** Draft | In Review | Approved
**Owner:** [name/role]
**Last updated:** [date]

## Summary

[2-4 sentences: what this design delivers and its overall shape — e.g. "an event-driven set of five services fronted by a public API gateway, replacing the monolithic claims module." No implementation deep-dive here, that belongs below.]

## Component Overview

| COMP-ID | Name | Responsibility | Satisfies (EPIC/REQ IDs) |
|---|---|---|---|
| COMP-001 | ... | ... | EPIC-001, REQ-FR-003 |

## Architecture Diagram

```mermaid
flowchart LR
    COMP001[Component Name] --> COMP002[Another Component]
```

## Component Details

### COMP-001 — [Name]

**Interfaces:** [summary-level — what it exposes/consumes, not a full spec]
**Data owned:** [what data/state this component is the source of truth for]
**Key dependencies:** [other COMP-IDs or external systems it relies on]

[repeat per component]

## NFR Mapping

| NFR ID | Responsible Component(s) | How Satisfied |
|---|---|---|
| REQ-NFR-001 | COMP-002 | ... |

## Data Flow & Integration Points

[Narrative description of how data moves through the system — request paths, async events, batch jobs — tying back to the diagram above.]

## ADRs Applied

| ADR ID | Decision | How This Design Respects It |
|---|---|---|

## Pre-Mortem

[Assume this design has already failed in production. The two or three most plausible reasons why, worked backward from the failure — not a restatement of Open Questions/Risks below.]

## Downstream Contract

| Consuming stage | What it reads from this document | What it's expected to give back |
|---|---|---|
| architecture-vs-requirements-checker | Full document | Pass/fail validation against requirements + NFRs |
| (add rows for any real downstream consumer outside the pipeline — engineering teams, a review board, etc.) | | |

## Open Questions / Risks

| # | Question/Risk | Why it matters | Owner |
|---|---|---|---|
```

Omit a section only if it's genuinely empty after a real attempt to populate it (e.g., no ADRs existed to apply). Never omit the frontmatter block, even when the rest of the document is short — it's what makes this file machine-readable by the next skill in the chain.

Save the file as `{slug}-architecture.md` and confirm the filename to the user, along with the note that `architecture-vs-requirements-checker` should validate it next against `{slug}-requirements.md` and `{slug}-nfrs.md`.

## Common failure modes to avoid

**Naming components after technology or layers instead of business responsibility.** "The microservices layer" or "the data layer" tells a reviewer nothing about what the component is *for*. If you can't write one clear sentence of business responsibility for a component, you've named an implementation detail, not a component.

**Leaving one or more NFRs unmapped to any component.** A performance, security, or compliance constraint that never made it into the NFR Mapping table is a gap this design will fail on later — find it during step 5, not after `architecture-vs-requirements-checker` finds it for you.

**Contradicting an accepted ADR without calling it out.** If your design naturally wants to diverge from a decision that's already `status: accepted`, that's a real conflict between the ADR and the requirements/NFRs driving your design — surface it explicitly in Open Questions rather than quietly designing around the ADR or quietly overriding it.

**Skipping the pre-mortem, or writing it as a restatement of Open Questions.** Open Questions/Risks captures what you already noticed while drafting. The pre-mortem is a separate, deliberately adversarial pass — assume failure already happened and work backward. If every pre-mortem item is also sitting in Open Questions verbatim, the pass wasn't run honestly.

**Over-specifying implementation detail that belongs in a later design phase.** Full database schemas, class diagrams, exact request/response payloads, and specific library choices are detailed design, not HLD. If you find yourself writing a column list or a JSON payload, step back up to what the component owns and exposes at a summary level instead.
