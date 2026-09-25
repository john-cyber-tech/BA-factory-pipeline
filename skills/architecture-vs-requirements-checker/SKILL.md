---
name: architecture-vs-requirements-checker
description: Cross-checks a solution architecture document against the requirements and non-functional requirements it's supposed to satisfy — verifying not just that every requirement and NFR maps to some component, but that the design as described is actually plausible for meeting stated NFR targets (a sub-200ms latency target routed through four synchronous, uncached service hops is a real red flag, not just a presence/absence checkbox). Use this whenever the user wants an architecture "validated," "checked against requirements," "reviewed for gaps," or asks "does this design actually meet our NFRs," "is this architecture ready to build from," or "did we miss anything in the design." This is the final quality gate in the BA-to-architecture pipeline — run it any time a solution architecture and its source requirements/NFRs both exist.
---

# Architecture vs Requirements Checker

## Pipeline contract

This skill is stage 11 — the final numbered stage — of an eleven-stage business-analysis-to-architecture pipeline. Each stage reads and/or writes a plain Markdown file carrying a small YAML frontmatter block, and one stage's output file is fed directly as the next stage's input file.

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
| 11 | **architecture-vs-requirements-checker** (you are here) | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

Two utility skills sit outside the numbered chain and can run at any point once relevant files exist: `traceability-matrix-generator` (reads whichever pipeline files exist, writes `{slug}-traceability-matrix.md`) and `domain-glossary-builder` (same inputs, writes `{slug}-glossary.md`).

`{slug}` is a short kebab-case initiative identifier carried in every document's frontmatter and reused across the whole chain. Gate stages (3, 6, 8, 11) route their findings back to the stage that produced the document under review — yours routes back to `solution-architecture-writer`. This is the last numbered stage in the main chain: there is no stage 12. After this gate, the initiative's design is either validated and ready to build from, or routed back to stage 10 for revision — either way, the numbered pipeline is done.

**Input contract for this skill:** expects three files — `{slug}-requirements.md` (`doc_type: requirements`), `{slug}-nfrs.md` (`doc_type: nfrs`), and `{slug}-architecture.md` (`doc_type: architecture`) — whose `initiative` fields all match. If `{slug}-nfrs.md` is missing, still perform the functional-coverage check, but flag prominently, in both the Summary and the NFR Coverage section, that NFR conformance and plausibility cannot be verified without it — don't quietly skip that half of the audit as if it passed. If `{slug}-requirements.md` or `{slug}-architecture.md` is missing, say so explicitly rather than assuming; you cannot run this check without both.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-architecture-review.md`, with the frontmatter block shown in Output Format below, including a machine-readable `review_verdict` field mirroring your Overall result. Tell the user the filename when done, and route them explicitly: if `review_verdict` isn't clean, the findings go back to `solution-architecture-writer` for revision; if it is clean, tell them the design is validated and ready to build from, and that `traceability-matrix-generator` can now produce a full end-to-end traceability view across the whole chain.

## Role

You're auditing the design against two sources of truth — the functional requirements and the non-functional requirements — not improving the architecture yourself. This is the failure mode you exist to catch: an architecture document that *looks* thorough, with a clean component diagram and confident naming, but that either drops a requirement silently, includes components nobody asked for (gold-plating or unvetted scope), or states NFR targets that don't actually match what the described design can plausibly deliver. A document can be internally consistent — every component well-named, every arrow labeled — and still fail the one thing that matters: does this design actually satisfy what the business asked for and what quality bar the system was told to hit.

Treat the requirements and NFR documents as the source of truth for *what's needed*. Treat the architecture document as a claim about *how that need will be met*. Your audit tests whether the claim holds up — including, uniquely at this stage, whether the claim is *believable* given what the design actually describes, not just whether the right words appear in the right table.

## What to check

**Functional coverage.** Every `REQ-BR-*` and `REQ-FR-*` ID in the requirements document should map to at least one component in the architecture that is explicitly responsible for it. Build this mapping ID by ID; don't eyeball it. A requirement with no owning component is a gap — it means nobody has actually designed for that piece of business need yet.

**NFR coverage.** Every `REQ-NFR-*` ID should map to a component with an explicit note on *how* it's satisfied — a caching layer, a redundancy strategy, an async queue, a specific data-partitioning approach. An NFR that appears in `{slug}-nfrs.md` but is absent from the architecture's NFR mapping entirely is a real gap, not a technicality — it means the design has no stated answer for that quality attribute at all.

**NFR plausibility.** This is the check unique to this stage, and the most important one to do carefully. For every NFR with a concrete, checkable target — a latency figure, an availability percentage, a throughput number, a recovery-time objective — don't just confirm the target is *mentioned*; read the actual path the architecture describes for that concern and sanity-check whether the design, as written, could plausibly hit it. A sub-200ms latency target routed through four synchronous, uncached service hops is implausible on its face. A 99.99% availability target with no stated redundancy, failover, or multi-zone strategy is implausible. A high-throughput target sitting behind a single unscaled component with no queueing or horizontal-scaling story is implausible. Flag these clearly as a **plausibility concern for a human architect to confirm** — you're reasoning from a document description, not measuring a running system, so phrase findings as "worth validating" rather than a certain failure. But flag them; a target that's merely present in a table next to a design that can't plausibly hit it is worse than an honestly-missing target, because it creates false confidence.

**Cost plausibility.** Apply the same discipline to every Cost & Resource Ceilings NFR (`nfr-elicitor`'s Cost category) as to latency or availability — a cost ceiling that's merely present in a table isn't the same as a design that can plausibly stay under it. Read the component(s) responsible for the cost-bearing operation and check for a stated control: a cap on tokens/calls per request, a caching or batching strategy that reduces call volume, a cheaper-model fallback or downgrade path, a rate limit, or a stated monitoring-plus-alert mechanism tied to the ceiling. A per-request LLM call in a hot path with no cache, no batching, no cheaper-model fallback, and a stated per-request cost ceiling is implausible on its face — nothing in the design explains how the ceiling gets enforced rather than just hoped for. A cost NFR with no component even attempting to control it is a straightforward coverage gap, not a plausibility judgment call — treat it as such. Where a control exists but its adequacy is a real unknown (e.g., a cache is mentioned but its expected hit rate isn't), phrase it as questionable, the same as any other plausibility finding here.

**Orphan components.** A component in the architecture with no `REQ-*` or `EPIC-*` driving its existence may be a legitimate infrastructure necessity (a message broker, a config service, an API gateway) or it may be unexplained scope that crept in during design. Say which it looks like for each one you find, and why — don't just list them as flags with no judgment attached.

**ADR conformance.** If any `{slug}-adr-*.md` files exist alongside these three inputs, check whether the architecture actually reflects each ADR's `accepted` decision, or quietly contradicts one — an architecture that routes around a decision it never revisited is a coordination failure worth surfacing even though ADRs aren't a required input for this skill.

**Component boundary sanity.** Flag any component whose stated responsibility is vague enough that two different engineers would plausibly build different things from it — the same ambiguity discipline `requirements-checker` applies to requirement statements, one level up the stack. A component description that just restates its name ("the Processing Service processes data") isn't a real boundary.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: architecture_review
schema_version: 1
initiative: <slug>
generated_by: architecture-vs-requirements-checker
source_files: [<requirements.md filename>, <nfrs.md filename>, <architecture.md filename>]
date: <today, ISO format>
review_verdict: architecture_sound | gaps_found | significant_gaps
---

# Architecture Review — [Initiative Name]

**Requirements source:** [doc/link]
**NFR source:** [doc/link, or "not supplied — NFR conformance unverified"]
**Architecture source:** [doc/link]
**Checked:** [date]
**Overall result:** Architecture sound — ready to build | Gaps found | Significant gaps — do not proceed to build

## Summary

[3-5 sentences: functional and NFR coverage rates, the most significant plausibility concerns if any, and a clear verdict on whether this architecture is safe to hand to delivery as-is. If `{slug}-nfrs.md` was missing, say so here explicitly.]

## Functional Coverage Matrix

| REQ-ID | Requirement (short) | Covered by Component | Status |
|---|---|---|---|
| REQ-BR-001 | ... | Order Service | OK |
| REQ-FR-004 | ... | — | **Gap** |

## NFR Coverage & Plausibility

| NFR-ID | Requirement | Mapped Component | Plausibility Assessment | Note |
|---|---|---|---|---|
| REQ-NFR-002 | p99 latency < 200ms | API Gateway → Pricing Service → Inventory Service (sync) | Questionable | Two synchronous hops with no cache layer stated; worth confirming with an architect against real p99 numbers |
| REQ-NFR-005 | 99.9% availability | Order Service | Plausible | Multi-AZ deployment and stated failover explicitly cover this |
| REQ-NFR-008 | LLM cost ≤ $0.02/request, p95 | Summarization Service | Questionable | Design calls the model per request with no stated cache or cheaper-model fallback; worth confirming expected call volume and per-call cost against the ceiling |

Plausibility Assessment values: **plausible** (design as described credibly supports the target), **questionable** (target may be achievable but the design omits a mechanism you'd expect to see — flag for architect confirmation), **implausible** (the described design and the stated target are hard to reconcile as written).

## Gaps

| Item | Type | Severity | Note |
|---|---|---|---|
| REQ-FR-004 | Functional gap | High | No component claims responsibility for this requirement |
| REQ-NFR-002 | Plausibility concern | Medium | See NFR table — sync chain vs. latency target |

Severity: **High** = a functional or business requirement, or an NFR with a hard compliance/contractual target, with zero coverage or a clearly implausible design. **Medium** = partial or questionable coverage, or a plausibility concern that needs an architect's confirmation rather than being a clear failure. **Low** = a minor ambiguity or an arguably out-of-scope item worth a sanity-check question.

## Orphan Components

| Component | Referenced by REQ/EPIC | Looks like | Recommendation |
|---|---|---|---|
| Audit Log Service | none found | Legitimate infra necessity | Confirm and backfill an NFR/requirement citing it, for traceability |

## ADR Conformance

[For each relevant ADR: does the architecture reflect its accepted decision, or contradict it? Omit this section if no ADR files were supplied.]

## Verdict

[One clear paragraph: is this architecture ready to build from as-is, or does it need another pass? If gaps or plausibility concerns exist, name the specific fix needed before proceeding — don't leave it vague.]
```

Keep `review_verdict` in the frontmatter consistent with **Overall result** in the body. Save the file as `{slug}-architecture-review.md` and tell the user the filename plus what to do next: if `review_verdict` isn't `architecture_sound`, hand the findings back to `solution-architecture-writer` for a revision; if it is, the initiative's design is validated and ready to build from, and `traceability-matrix-generator` can now produce a full end-to-end traceability view across the entire chain.

## Calibration notes

Do the ID-by-ID mapping exhaustively for both functional coverage and NFR coverage — this is a mechanical cross-reference check first, and skipping requirements because they "obviously" must be handled somewhere is exactly how gaps slip through in real reviews.

Plausibility findings are judgment calls, not hard facts. You're reasoning about system behavior from a document description, not running the system or measuring anything — so phrase every plausibility concern as something "worth confirming with an architect" rather than a definitive verdict, and reserve "implausible" for cases where the described design and the stated target are genuinely hard to reconcile on their face (a strict latency target with a long uncached synchronous chain, a high-availability target with no stated redundancy at all, a per-request cost ceiling with an uncached per-request model call and no fallback). Don't manufacture plausibility concerns where the architecture document simply didn't spell out every mechanism — distinguish "this design can't plausibly hit this target" from "this document doesn't say enough for me to tell," and say which one you mean. This applies equally to cost ceilings — they're a plausibility judgment, not a pass/fail checkbox, same as latency or availability.

If any of the three required input files is missing, say so plainly and explain which specific check is blocked as a result, rather than skipping it silently or guessing at content that isn't there. A functional-only review with NFR conformance explicitly marked unverified is a legitimate, honest output; a review that quietly treats missing NFRs as passing is not.
