---
name: requirements-writer
description: Turns messy input — stakeholder notes, meeting transcripts, emails, a rough feature idea, an existing bullet list — into well-formed, testable business and functional requirements, formatted as a structured Confluence-style page. Use this whenever the user wants to "write requirements," "document what the business needs," "turn these notes into a BRD/requirements doc," "formalize this feature ask," or hands over raw stakeholder input and wants it turned into something a delivery team could actually build from. Also trigger when a requirements document already exists but reads vague, unstructured, or untestable and the user wants it rewritten properly — this skill knows the difference between a real requirement and a wish. Push to use this any time "requirements," "BRD," "FRD," "business need," or "spec" comes up in a business-analysis context, even if the user just pastes raw notes without using those words.
---

# Requirements Writer

## Pipeline contract

This skill is stage 2 of an eleven-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file — no reformatting, no copy-pasting between tools, no losing traceability at a handoff.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1 | stakeholder-discovery | raw notes, interview transcripts, workshop notes — freeform | `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) |
| 2 | **requirements-writer** (you are here) | `{slug}-stakeholder-needs.md`, or raw input if stage 1 was skipped | `{slug}-requirements.md` (`doc_type: requirements`) |
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

`{slug}` is a short kebab-case identifier for the initiative (e.g. `claims-portal-refresh`) that stays constant across every document in the chain — it's how a human or a script can tell which files belong together. Gate stages (3, 6, 8, 11) typically drive a revision loop back to the stage that produced the document under review (2, 5, 7, 10 respectively), rather than flowing straight downstream — keep that in mind when you present results.

**Input contract for this skill:** your primary input is `{slug}-stakeholder-needs.md` (`doc_type: stakeholder_needs`) if stage 1 ran — read its Needs table and trace each requirement you write back to the NEED-ID(s) it addresses where possible. If stage 1 was skipped, your input is freeform — notes, a transcript, a rough ask — and that's fine too, structuring it is the job either way. If instead you're handed a file with `doc_type: requirements` frontmatter, you're revising an existing document, not starting fresh: keep every existing REQ-ID stable (see step 5 below), and set `source_file` in your output frontmatter to that file's name so the revision history stays traceable. Pick `{slug}` from whichever input file's `initiative` field is present.

**Output contract for this skill:** always write your result to an actual `.md` file (don't just print it in the chat) named `{slug}-requirements.md`, with the frontmatter block shown in Output Format below. When you're done, tell the user the filename and that it's ready to hand to `requirements-checker` (quality pass), `nfr-elicitor` (deepen non-functional coverage), or straight to `requirements-to-epics`.

## What makes a requirement worth writing

A requirement earns its place in the document if two different engineers reading it independently would build the same thing, and a tester reading it could write a pass/fail test without asking the author what they meant. Most requirements documents fail this test not because the author didn't know the domain, but because they wrote down the *solution* ("add a button that emails the manager") instead of the *need* ("the manager must be notified within 5 minutes of a request being submitted"), or because they wrote something true but unfalsifiable ("the system should be fast," "the UI should be intuitive").

Your job with this skill is to take whatever raw material the user gives you — transcripts, bullet points, a Slack thread, a half-formed idea — and produce requirements that are specific enough to build from, testable enough to verify, and traceable enough that later skills (epic breakdown, architecture) can point back to exactly which requirement drove which decision.

## Process

1. **Read everything first.** Don't start drafting requirement-by-requirement as you skim. Read the full input, then identify: who the stakeholders/actors are, what business outcome they're after, what's explicitly in scope, and what's implied but unstated. Raw notes usually bury 2-3 real requirements inside a paragraph of context and opinion — separate the signal from the narrative.

2. **Ask before assuming, when it's cheap to ask.** If the input is ambiguous on something that would materially change what gets built (e.g., "notify the manager" — email? SMS? in-app? all three?), flag it as an open question rather than silently picking one. If you're working unattended or the answer is a reasonable, low-stakes default, state your assumption inline in the requirement's notes rather than blocking on it.

3. **Classify each requirement** as one of:
   - **BR (Business Requirement)** — a business goal or outcome, technology-agnostic. "Reduce average claim processing time to under 24 hours."
   - **FR (Functional Requirement)** — a specific system behavior. "The system shall reject a claim submission if the required documents field is empty."
   - **NFR (Non-Functional Requirement)** — a quality constraint: performance, security, availability, compliance, usability. "The claims API shall respond within 500ms at p95 under expected load." Don't skip these — a requirements doc with zero NFRs is a common gap that architecture-stage skills will trip over later. If the input doesn't mention any, ask the user whether known constraints exist (regulatory, SLA, existing platform limits) rather than inventing numbers. This table only needs to capture what's already evident — a full, checklist-driven elicitation pass is `nfr-elicitor`'s job (stage 4), not yours.

4. **Write each requirement to be atomic and testable.** One requirement, one testable statement. If you catch yourself writing "and" to join two unrelated behaviors, split it into two requirements. Use "shall" for mandatory behavior — it's a convention that makes the mandatory/optional distinction scannable at a glance, not just formality for its own sake.

5. **Give every requirement a stable ID**: `REQ-BR-001`, `REQ-FR-001`, `REQ-NFR-001`, numbered sequentially within each type. These IDs are the backbone of traceability — the requirements-to-epics and epics-checker skills key off them, so don't renumber existing IDs when revising a document; append new ones.

6. **Attach acceptance criteria to functional requirements.** A functional requirement without a way to verify it is just a sentence. Use Given/When/Then where the behavior is conditional, or a plain checklist where it's simpler than that.

   **If the requirement describes AI/ML/non-deterministic behavior** (a model output, a classification, a generated response, a ranking, anything that won't return the exact same result twice), a Given/When/Then or checklist AC is not enough — the behavior can't be pinned to an exact expected value. Instead, the AC must name three things:
   - **Metric** — what's actually measured (e.g. Macro F1, Precision@5, refusal rate, human-eval pass rate)
   - **Threshold** — the specific number or range that counts as passing
   - **Enforcement type** — **Gate** (blocks release/build in CI if missed) or **Monitor** (tracked in production, doesn't block release)

   "The classifier should be accurate" is not acceptable. "The ticket-tagging classifier shall achieve Macro F1 ≥ 90% against the golden validation set `support_eval_v2.json`, enforced as a CI gate" is. If you don't know the threshold yet, write the metric and enforcement type, flag the threshold itself as an Open Question — don't invent a number, and don't skip the AC because the number isn't settled.

## Quality checklist (apply to every requirement before finalizing)

- **Atomic** — describes one thing, not several joined by "and"/"or"
- **Testable** — a specific, observable pass/fail check exists
- **Unambiguous** — no words like "fast," "easy," "intuitive," "robust," "flexible," "accurate," "correct" without a number or concrete definition attached
- **Eval-gated (AI/ML requirements only)** — if the requirement describes non-deterministic behavior, its AC names a metric, a threshold, and Gate/Monitor — not just a qualitative description of desired behavior
- **Solution-free** (for BRs/FRs describing need, not implementation) — describes *what*, not *how*, unless the how is itself the requirement
- **Traceable** — has an ID and, where relevant, a link back to the stakeholder need or source note it came from
- **Owned** — has a named or role-based owner/source, not just "the business"

If a requirement fails one of these, fix it before including it — don't include known-bad requirements with a note to fix later; the document should be usable as-is.

## Output format

Produce a single Markdown **file** — the frontmatter below plus a Confluence-style body (headers, tables, no nested bullets more than one level deep — this renders cleanly whether it's opened as a file, pasted into Confluence, or read back in by the next skill). Use exactly this structure, frontmatter included:

```markdown
---
doc_type: requirements
schema_version: 1
initiative: <slug>
generated_by: requirements-writer
source_file: <input filename if you revised an existing requirements.md, else null>
date: <today, ISO format>
id_prefixes_used: [REQ-BR, REQ-FR, REQ-NFR]
---

# [Feature/Initiative Name] — Requirements

**Status:** Draft | In Review | Approved
**Owner:** [name/role]
**Stakeholders:** [list]
**Last updated:** [date]
**Source material:** [what this was derived from — meeting notes, transcript, prior doc, etc.]

## Summary

[2-4 sentences: what business problem this solves and for whom. No jargon a stakeholder outside the immediate team wouldn't understand.]

## Scope

**In scope:** [bullet list]
**Out of scope:** [bullet list — explicitly naming what's excluded prevents scope disputes later]

## Business Requirements

| ID | Requirement | Rationale/Source |
|---|---|---|
| REQ-BR-001 | ... | ... |

## Functional Requirements

| ID | Requirement | Acceptance Criteria | AI/ML? | Source |
|---|---|---|---|---|
| REQ-FR-001 | The system shall ... | Given ... When ... Then ... | No | REQ-BR-001 |
| REQ-FR-002 | The system shall ... | Metric: ... · Threshold: ... · Enforcement: Gate/Monitor | Yes | REQ-BR-001 |

## Non-Functional Requirements

| ID | Category | Requirement | Target/Metric |
|---|---|---|---|
| REQ-NFR-001 | Performance | ... | ... |

## Open Questions

| # | Question | Why it matters | Owner |
|---|---|---|---|

## Assumptions

[Anything you had to assume to write a concrete requirement — list explicitly so reviewers can challenge it.]
```

Omit a section only if it's genuinely empty after a real attempt to populate it (e.g., no open questions) — don't pad sections with filler to make the doc look complete. Never omit the frontmatter block, even when the rest of the document is short — it's what makes this file machine-readable by the next skill in the chain.

Save the file as `{slug}-requirements.md` and confirm the filename to the user, along with which skill(s) can consume it next (`requirements-checker`, `requirements-to-epics`).

## Common failure modes to avoid

**Writing requirements as tasks.** "Build a login page" is a task, not a requirement. The requirement is what the login page must do or enforce: "Users shall be locked out after 5 consecutive failed login attempts within 10 minutes."

**Confusing a requirement with a UI decision.** "Add a dropdown for status" bakes in a UI choice the business didn't actually ask for. Prefer "Users shall be able to select one of [defined set] statuses" and let a design/architecture stage decide it's a dropdown.

**Silently resolving genuine ambiguity.** If two stakeholders in the source material seem to want contradictory things, surface that conflict in Open Questions rather than picking a side.

**Padding with restated obvious requirements** to make the doc look thorough (e.g., "the system shall allow users to log in" for a feature that isn't about login at all). Every requirement should trace to an actual need in the source material or an explicit, flagged assumption.
