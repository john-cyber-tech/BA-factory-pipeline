# BA-to-Architecture Skill Pipeline — Reference Documentation

**Version:** Phase 1 (AI Factory enhancement pass)
**Scope:** 13 skills, 11 numbered pipeline stages + 2 utility skills
**Audience:** This document is written to be sufficient, on its own, for an LLM agent or a human operator to run any stage of this pipeline correctly without reading the individual `SKILL.md` files first. The `SKILL.md` files remain the authoritative source for exact wording and edge-case handling; this README is the map, the contract, and the operating manual.

---

## 1. What this pipeline is

This is a chain of 13 specialist skills that take an initiative from raw discovery material (interview notes, a rough feature idea) through to a validated high-level solution architecture, with a quality gate after every authoring stage and two utility skills that can run at any point once enough files exist.

Each skill does exactly one job — author one type of document, or critique one type of document against its source — and hands off to the next skill through a plain Markdown file on disk, not through conversation state. This means:

- Any stage can be run in a fresh session with no memory of prior stages, as long as the right input file(s) are provided.
- The pipeline can be paused indefinitely between any two stages.
- A human can intervene at any point — edit a file by hand, skip a stage, or re-run a stage — without breaking the chain, as long as the file-naming and frontmatter conventions (Section 3) are preserved.
- Multiple initiatives can be in flight simultaneously in the same working directory, distinguished only by their `{slug}`.

## 2. Operating assumptions and dependencies

**Read this section before invoking any skill.** These are the assumptions every skill in the pack is written against. If any of them don't hold for your environment, the skill's behavior at the edges is not guaranteed.

- **File system access.** Every skill assumes it can read and write plain `.md` files in a shared working directory (a repo, a local folder, a mounted drive — the skills are agnostic to which). A skill that cannot write an actual file should not be considered to have completed its job; printing output to chat only is not equivalent to producing the artifact.
- **No inter-skill memory.** No skill assumes it has seen the output of any other skill's conversation — only its output *file*. Do not rely on conversational context carrying information from stage to stage; if a fact matters, it must be in the file.
- **`{slug}` is chosen once, at the start of the chain**, by whoever runs `stakeholder-discovery` (or by whoever runs `requirements-writer` directly, if stage 1 is skipped). It is a short, kebab-case identifier for the initiative (e.g. `claims-portal-refresh`) and must be reused, unchanged, in every document produced for that initiative. Every skill in this pack reads `{slug}` from an input file's `initiative` frontmatter field when one is available, rather than inventing a new one.
- **IDs are permanent once assigned.** `NEED-*`, `REQ-BR-*`, `REQ-FR-*`, `REQ-NFR-*`, `EPIC-*`, `STORY-*`, `ADR-*`, `COMP-*` — once an ID is written into a document, it is never renumbered or reused, even across revisions. Revising a document means extending its ID sequence, not restarting it. This is what keeps cross-document traceability intact; renumbering silently breaks every downstream reference to that ID.
- **Authors don't grade their own work.** Every authoring skill (writes a new artifact) has a paired checker skill (audits an existing artifact against its source). Checkers do not rewrite content — they diagnose and route findings back to the authoring skill. Don't ask a checker skill to "just fix it"; that's a different skill's job, working from the checker's findings.
- **No fabrication, ever.** Every skill in this pack is written with an explicit instruction not to invent facts, numbers, owners, or links that aren't supported by the source material — an honest gap, flagged as an Open Question, is treated as a correct and complete output; a fabricated answer is treated as a defect even if it looks more complete. This is the single most load-bearing convention in the whole pack — an LLM operating any of these skills should treat "I don't know, and here's what would resolve it" as a fully acceptable deliverable for any individual field.
- **Markdown, not prose-only.** Every output is a structured Markdown file with a YAML frontmatter block and a Confluence-style body (headers, tables, at most one level of nested bullets). This isn't a style preference — it's what makes the file mechanically parseable by the next skill and by the two utility skills, which scan multiple documents' frontmatter and tables programmatically-in-effect.
- **English domain terminology, no assumed industry.** The skills carry no built-in assumption about what industry or domain the initiative is in. Domain vocabulary, regulatory regimes, and technical constraints must come from the source material or be raised as an explicit question — never assumed from a "typical" project in that space.
- **No external tool calls required.** Every skill in this pack can be executed by an LLM with nothing more than read/write access to the working directory and the ability to reason over the text it's given. None of them require web search, code execution, or a database — if an LLM operating this pipeline reaches for a tool beyond file I/O, that's a sign it's stepped outside the skill's intended scope.

## 3. File and frontmatter conventions

Every document in the pipeline shares a frontmatter schema. The fields that appear across every stage:

```yaml
doc_type: <one of the registry values below>
schema_version: 1
initiative: <slug>
generated_by: <skill name that produced this file>
source_file(s): <filename(s) this was derived from, or null>
date: <ISO 8601 date>
```

Stage-specific fields (an ID-prefix registry, a `review_verdict`, an `epic_id`, a `decision_id`, etc.) are documented per-skill in Section 5.

### `doc_type` registry

| `doc_type` | Produced by | Filename pattern |
|---|---|---|
| `stakeholder_needs` | `stakeholder-discovery` | `{slug}-stakeholder-needs.md` |
| `requirements` | `requirements-writer` | `{slug}-requirements.md` |
| `requirements_review` | `requirements-checker` | `{slug}-requirements-review.md` |
| `nfrs` | `nfr-elicitor` | `{slug}-nfrs.md` |
| `epics` | `requirements-to-epics` | `{slug}-epics.md` |
| `epics_review` | `epics-vs-requirements-checker` | `{slug}-epics-review.md` |
| `stories` | `epic-to-story-decomposer` | `{slug}-epic-<EPIC-ID>-stories.md` |
| `stories_review` | `story-checker` | `{slug}-epic-<EPIC-ID>-stories-review.md` |
| `adr` | `adr-writer` | `{slug}-adr-<ADR-ID>-<short-name>.md` |
| `architecture` | `solution-architecture-writer` | `{slug}-architecture.md` |
| `architecture_review` | `architecture-vs-requirements-checker` | `{slug}-architecture-review.md` |
| `traceability_matrix` | `traceability-matrix-generator` | `{slug}-traceability-matrix.md` |
| (glossary — no fixed `doc_type` key beyond its own frontmatter) | `domain-glossary-builder` | `{slug}-glossary.md` |

### ID prefix registry

| Prefix | Assigned by | Meaning |
|---|---|---|
| `NEED-XXX` | `stakeholder-discovery` | A discrete stakeholder need, solution-free |
| `REQ-BR-XXX` | `requirements-writer` | Business requirement |
| `REQ-FR-XXX` | `requirements-writer` | Functional requirement |
| `REQ-NFR-XXX` | `requirements-writer` (thin set) / `nfr-elicitor` (authoritative set) | Non-functional requirement |
| `EPIC-XXX` | `requirements-to-epics` | An epic — a coherent, independently valuable grouping of requirements |
| `STORY-XXX` | `epic-to-story-decomposer` | A sprint-sized story within one epic |
| `ADR-XXX` | `adr-writer` | One architecture decision record |
| `COMP-XXX` | `solution-architecture-writer` | One architecture component |
| `F-XXX` | `requirements-checker` | A finding in a requirements review (not a pipeline entity — local to the review document) |

## 4. Pipeline map

```
1  stakeholder-discovery
        |  {slug}-stakeholder-needs.md
        v
2  requirements-writer  <------------------.
        |  {slug}-requirements.md          | (revise)
        v                                  |
3  requirements-checker --------------------'   [GATE]
        |  {slug}-requirements-review.md
        v (if ready)
   +----+----+
   v         v
4  nfr-elicitor      5  requirements-to-epics  <------.
   | {slug}-nfrs.md      | {slug}-epics.md            |
   +--------+------------+                            | (revise)
            v                                          |
   6  epics-vs-requirements-checker --------------------'   [GATE]
            |  {slug}-epics-review.md
            v (if full_coverage, per epic)
   7  epic-to-story-decomposer  <------------.
            |  {slug}-epic-<ID>-stories.md   | (revise)
            v                                |
   8  story-checker -----------------------  '   [GATE]
            |  {slug}-epic-<ID>-stories-review.md
            v (if ready)
   9  adr-writer (runs once per architecturally significant decision)
            |  {slug}-adr-<ID>-<name>.md
            v
  10  solution-architecture-writer  <-----.
            |  {slug}-architecture.md     | (revise)
            v                             |
  11  architecture-vs-requirements-checker '   [GATE - final numbered stage]
            |  {slug}-architecture-review.md
            v (if architecture_sound)
       Ready to build.

  Utility (run any time >=2 pipeline files exist, and re-run as more accumulate):
    traceability-matrix-generator -> {slug}-traceability-matrix.md
    domain-glossary-builder       -> {slug}-glossary.md
```

**Gate stages** (3, 6, 8, 11) do not flow forward automatically. Each writes a `review_verdict` (or equivalent) that a human — or the orchestrating process — reads to decide whether to proceed downstream or route back to the authoring stage for revision. No skill in this pack unilaterally decides to proceed past a failed gate; that decision belongs to whoever is operating the pipeline.

## 5. Skill catalog

Each entry below covers: **Purpose**, **Trigger phrases** (what a request should sound like to invoke this skill), **Inputs**, **Outputs**, **Core process**, **Key rules**, **Common failure modes**, and — for the 8 skills touched in this Phase 1 pass — **What changed and why**.

---

### 5.1 `stakeholder-discovery` — Stage 1 (entry point)

**Purpose.** Turns raw discovery material — interview notes, workshop transcripts, kickoff meeting minutes, a loose list of "who wants what" — into a structured stakeholder needs document: who the stakeholders are, what each needs and why, a RACI, and a set of discrete, solution-free need statements.

**Trigger phrases.** "stakeholder discovery," "who are our stakeholders," "capture these interview notes," "process these workshop/kickoff notes," "build stakeholder needs / a RACI," "who wants what here."

**Inputs.** No required upstream file — this is the pipeline's entry point. Input is typically freeform text (transcripts, notes, emails). If handed a file already carrying `doc_type: stakeholder_needs`, treat it as a revision task, not a fresh start.

**Outputs.** `{slug}-stakeholder-needs.md` — Stakeholders table, RACI, Needs table (`NEED-XXX`), Conflicting/Competing Needs, Open Questions.

**Core process.** Read all raw material fully before extracting anything → capture role/needs/pain-points/influence per stakeholder → build a RACI only where the material actually supports it (leave cells blank rather than invent ownership) → extract each need as a discrete, solution-free, stably-ID'd line → actively surface conflicting needs rather than resolving or blending them → deliberately check for quiet/low-power stakeholders the material barely mentions.

**Key rules.** A need describes the underlying problem/want, never a system behavior ("the system shall...") — that's `requirements-writer`'s job. Needs and RACI cells are allowed to be thin or blank; a fabricated RACI is treated as worse than an honest gap.

**Common failure modes.** Writing needs as requirements; fabricating RACI/influence ratings; letting one dominant interview voice drown out briefly-mentioned stakeholders; smoothing over real disagreement into a mushy compromise statement.

**Downstream consumer.** `requirements-writer` (stage 2), which traces requirements back to `NEED-*` IDs.

---

### 5.2 `requirements-writer` — Stage 2

**Purpose.** Turns messy input (or a stakeholder-needs document) into well-formed, testable business and functional requirements.

**Trigger phrases.** "write requirements," "document what the business needs," "turn these notes into a BRD/FRD," "formalize this feature ask." Also triggers on an existing but vague/untestable requirements doc that needs rewriting.

**Inputs.** `{slug}-stakeholder-needs.md` (preferred — trace requirements back to `NEED-*` IDs) or raw freeform input if stage 1 was skipped. If handed a `doc_type: requirements` file, treat as revision (keep existing `REQ-*` IDs stable, append new ones).

**Outputs.** `{slug}-requirements.md` — Scope (in/out), Business Requirements (`REQ-BR-*`), Functional Requirements (`REQ-FR-*`, with an **AI/ML?** column — see 5.9), Non-Functional Requirements (a thin starter set; full elicitation is `nfr-elicitor`'s job), Open Questions, Assumptions.

**Core process.** Read everything before drafting → ask (or flag as Open Question) rather than silently resolve material ambiguity → classify each requirement as BR/FR/NFR → write atomic, testable, single-behavior requirements using "shall" for mandatory behavior → assign stable sequential IDs per type → attach acceptance criteria to every FR, in Given/When/Then form for deterministic behavior **or** the eval-gated metric/threshold/enforcement form for AI/ML behavior (Section 5.9).

**Key rules.** A requirement earns its place if two engineers reading it independently would build the same thing, and a tester could write a pass/fail check without a follow-up question. Solution-free unless the "how" genuinely is the requirement. Every requirement traceable to a need or a flagged assumption.

**Common failure modes.** Writing tasks instead of requirements ("build a login page" vs. what it must enforce); baking in UI decisions; silently resolving stakeholder contradictions instead of surfacing them; padding with obvious/irrelevant requirements to look thorough.

**Downstream consumers.** `requirements-checker` (quality gate), `nfr-elicitor` (deepen NFR coverage), `requirements-to-epics` (grouping).

---

### 5.3 `requirements-checker` — Stage 3 [GATE]

**Purpose.** Audits an existing requirements document for ambiguity, untestability, duplication, missing AC, and NFR gaps. A critic, not an author — does not rewrite.

**Trigger phrases.** "review/quality-check/validate/sanity-check these requirements," "are these ready for the team," "what's wrong with this."

**Inputs.** A file with `doc_type: requirements`. If the input doesn't look like a requirements document at all, say so and redirect to `requirements-writer` rather than forcing a review.

**Outputs.** `{slug}-requirements-review.md` — per-requirement Findings table (with severity), Document-Level Gaps, Duplicate/Conflicting Requirements, Scorecard (1–5 across five dimensions), and a machine-readable `review_verdict`: `ready` | `ready_with_minor_fixes` | `needs_rework`.

**Core process.** Per requirement: testability, ambiguity (flag vague qualifiers with no attached number/definition), atomicity, solutioning-vs-need, duplication/conflict, traceability, missing AC. Across the document: NFR coverage gaps, scope clarity, terminology consistency.

**Severity scale.** Blocker (cannot build/test as written) → Major (will cause rework/disagreement) → Minor (polish) → Gap (something missing entirely, not something wrong).

**Common failure modes.** Manufacturing findings to look thorough on a genuinely clean document; softening real blockers to be polite.

**Routing.** Not forward-flowing — routes back to `requirements-writer` if `review_verdict` isn't `ready`.

**What changed and why (Phase 1).** Added an explicit **AI/ML acceptance criteria shape** check: for requirements marked (or plainly describing) non-deterministic behavior, the correct AC shape is metric + threshold + Gate/Monitor, not Given/When/Then, and the checker now evaluates each requirement against the bar that actually applies to it rather than uniformly expecting Given/When/Then. This closes a gap created by the Phase 1 change to `requirements-writer` (5.9) — without this update, the checker would have flagged correctly-formed eval-gated AC as defective, or passed AI/ML requirements with no real testability check at all.

---

### 5.4 `nfr-elicitor` — Stage 4

**Purpose.** Runs a systematic, checklist-driven elicitation pass over non-functional requirements, forcing an explicit answer — a real target or an honest "no constraint identified" — for every category, rather than letting NFRs silently go unmentioned.

**Trigger phrases.** "elicit NFRs," "non-functional requirements," "quality attributes," "SLAs/constraints," "flesh out the NFRs." Also triggers proactively when a requirements doc's NFR coverage looks thin.

**Inputs.** `{slug}-requirements.md` (its thin NFR table is the starting set, not the finished product) and, if available, `{slug}-stakeholder-needs.md` for constraints stakeholders mentioned but that never made it into a formal NFR line. If handed a `doc_type: nfrs` file, treat as a continuation pass.

**Outputs.** `{slug}-nfrs.md` — NFR Catalog (`REQ-NFR-*`, with Verification Method and Relevant Epics columns), a **13-category** Coverage Checklist, Conflicts/Tensions, Open Questions.

**The 13 categories.** Performance & Latency, Throughput & Capacity, Availability & Uptime/DR, Scalability, Security & Access Control, Data Privacy & Regulatory Compliance, Accessibility, Browser/Device/Platform Support, Auditability & Logging, Observability & Monitoring, Error Handling & Recovery, Data Retention & Archival, **Cost & Resource Ceilings**.

**Core process.** Pull forward existing `REQ-NFR-*` entries unchanged → work every category to an explicit answer (never omit a category row) → attach a verification method to every real target → actively hunt for NFR-to-NFR conflicts and record them rather than resolving them → never fabricate a plausible-sounding number.

**What changed and why (Phase 1).** Added **Cost & Resource Ceilings** as the 13th category — covering LLM/inference token spend, third-party API cost, compute/storage spend at scale, and model-downgrade/throttling ceilings. This exists because AI-touching features have cost profiles that scale with usage in ways ordinary CRUD features don't; "no constraint identified" is a materially riskier answer here than for most other categories, and prior to this change the checklist had no line item forcing the question to even be asked.

**Once this file exists, it is the authoritative NFR source** — every downstream skill should prefer it over the thin table still sitting in `{slug}-requirements.md`.

---

### 5.5 `requirements-to-epics` — Stage 5

**Purpose.** Groups a requirement set into coherent, independently valuable epics, with rationale, suggested sequencing, and full traceability back to every source requirement.

**Trigger phrases.** "turn requirements into epics," "break this down for the backlog," "what epics come out of this," "what are the workstreams here."

**Inputs.** `{slug}-requirements.md` (required) and `{slug}-nfrs.md` (preferred — use as the authoritative NFR source when present). If a `{slug}-requirements-review.md` exists with a non-`ready` verdict, flag that before proceeding.

**Outputs.** `{slug}-epics.md` — per-epic (`EPIC-XXX`): delivers-statement, rationale, requirements covered, relevant NFRs, T-shirt size, dependencies; plus a Suggested Sequence and a Coverage Check table.

**Core process.** Read every requirement before grouping → group by natural seams (shared actor/entity/process step), not superficial similarity → name epics for the capability delivered, not the implementation mechanism → one paragraph of rationale per epic → suggest a dependency-aware sequence → size relatively (S/M/L/XL) → confirm every `REQ-ID` lands in at least one epic or is explicitly excluded with a reason.

**Common failure modes.** Grouping by document section instead of capability; losing NFRs in the shuffle; sizing by requirement count instead of real complexity; silently narrowing scope instead of flagging an unplaceable requirement.

**Downstream consumers.** `epics-vs-requirements-checker` (coverage gate), `epic-to-story-decomposer` (per epic).

---

### 5.6 `epics-vs-requirements-checker` — Stage 6 [GATE]

**Purpose.** Traceability audit between a requirements document and its epic breakdown — verifies the mapping is complete, accurate, and honest. Does not author or improve epics.

**Trigger phrases.** "check these epics against requirements," "did we miss anything," "is this epic breakdown complete."

**Inputs.** `doc_type: requirements` + `doc_type: epics` files with matching `initiative`, plus `{slug}-nfrs.md` if present (authoritative for the NFR coverage check).

**Outputs.** `{slug}-epics-review.md` — coverage findings, `review_verdict`: `full_coverage` | `gaps_found` | `significant_gaps`.

**Core process.** Full coverage (every `REQ-ID` maps to ≥1 epic, ID by ID) → no silent exclusions (every stated exclusion has a reason) → orphan epic content (scope in an epic with no `REQ-ID` behind it) → duplicate primary-ownership coverage → NFR coverage specifically (most likely category to get lost) → sequencing sanity against stated dependencies.

**Routing.** Routes back to `requirements-to-epics` if not `full_coverage`.

---

### 5.7 `epic-to-story-decomposer` — Stage 7

**Purpose.** Breaks one epic (or a full breakdown) into sprint-sized, INVEST-compliant stories with Given/When/Then acceptance criteria and full traceability.

**Trigger phrases.** "break an epic into stories," "write user stories for this," "get this ready for sprint planning/grooming," "how do we slice this epic."

**Inputs.** A `doc_type: epics` file. If a `{slug}-epics-review.md` exists with a verdict other than `full_coverage`, flag that before proceeding.

**Outputs.** One file per epic: `{slug}-epic-<EPIC-ID>-stories.md` — per-story (`STORY-XXX`): "As a / I want / so that," traces-to, size, dependencies, Acceptance Criteria (Given/When/Then, plus an AI/ML block where applicable — see 5.9), Sequencing Notes, Coverage Check.

**Core process.** Read the epic's full context and pull forward its requirements' existing AC where available → identify distinct user-observable behaviors (not architecture layers) → write each story in the standard three-part form, with a real "so that" benefit → apply INVEST explicitly, letter by letter → write Given/When/Then AC for the happy path and the significant edge/error cases → for AI/ML behavior, carry the metric/threshold/enforcement shape down instead of forcing Given/When/Then (5.9) → trace every story to its epic and requirement(s) → confirm full coverage before finishing.

**Common failure modes.** Slicing by technical layer (backend story / frontend story); AC that just restates the story title; over-slicing near-duplicate stories; forcing a cross-cutting NFR into a fake standalone story; forcing an AI/ML story's behavior into an uncheckable Given/When/Then.

**What changed and why (Phase 1).** Step 5 and the output template now explicitly branch: deterministic behavior gets Given/When/Then; non-deterministic behavior (traced to an AI/ML-flagged requirement, or plainly non-deterministic even if untagged) gets the inherited metric/threshold/enforcement AC instead — the two forms can coexist within one story when it has both kinds of behavior. This closes the gap where a story delivering, say, a model-generated summary would otherwise have been forced into an unfalsifiable "given the user asks, when the model responds, then the response is good" AC.

---

### 5.8 `story-checker` — Stage 8 [GATE]

**Purpose.** Audits a story breakdown against INVEST, acceptance-criteria quality, and epic-traceability. A critic, mirroring `requirements-checker` one level down.

**Trigger phrases.** "review these stories," "are these ready for grooming," "is this backlog INVEST-compliant."

**Inputs.** A `doc_type: stories` file; ideally also the `{slug}-epics.md` it traces to (if absent, traceability checks are limited but INVEST/AC-quality checks still run in full).

**Outputs.** `{slug}-epic-<EPIC-ID>-stories-review.md` — findings by story and INVEST dimension, Coverage Check, Slicing Concerns, Scorecard, `review_verdict`: `ready` | `ready_with_minor_fixes` | `needs_rework`.

**Core process.** Vertical-slicing check → INVEST letter by letter, per story → acceptance-criteria quality (flag title-restating AC, missing edge cases, non-checkable prose) → traceability both directions (story → epic/requirement, and epic scope → some story) → over-/under-slicing → NFR handling (flag fake standalone NFR "stories").

**Routing.** Routes back to `epic-to-story-decomposer` if not `ready`.

**What changed and why (Phase 1).** The **Testable** INVEST check and the **AC quality** check now explicitly accept the metric/threshold/enforcement shape as valid for AI/ML stories, instead of only recognizing Given/When/Then as checkable. The checker also now actively flags the *inverse* failure: a story that plainly delivers non-deterministic behavior but was written with only Given/When/Then and no metric/threshold/enforcement anywhere — treated as the decomposer having missed the pattern, at the same severity a missing AC block would get on an ordinary story.

---

### 5.9 Cross-cutting pattern: eval-gated acceptance criteria for AI/ML requirements

Introduced at `requirements-writer` (5.2) and propagated through `requirements-checker` (5.3), `epic-to-story-decomposer` (5.7), and `story-checker` (5.8). Documented once here because all four skills must agree on it for the pipeline to stay internally consistent.

**The problem.** A Given/When/Then acceptance criterion assumes a fixed expected output for a fixed input. That assumption breaks for anything genuinely non-deterministic — a model output, a classification, a generated response, a ranking. "The recommendations should be relevant" is not falsifiable; a tester cannot write a pass/fail check from it, and an AI agent building against it has no target to converge on.

**The rule.** Any functional requirement or story describing AI/ML/non-deterministic behavior must state three fields instead of (or in addition to) Given/When/Then:

- **Metric** — what's actually measured (e.g., Macro F1, Precision@5, refusal rate, human-eval pass rate, ROUGE-L)
- **Threshold** — the specific number or range that counts as passing
- **Enforcement type** — **Gate** (blocks the build/release in CI if missed) or **Monitor** (tracked in production, doesn't block release)

Example: *"The ticket-tagging classifier shall achieve Macro F1 ≥ 90% against the golden validation set `support_eval_v2.json`, enforced as a CI gate."*

**What each skill in the chain is responsible for:**
- `requirements-writer` identifies AI/ML requirements (an explicit AI/ML? column) and writes the three fields; if the threshold isn't yet known, it states the metric and enforcement type and raises the threshold as an Open Question rather than inventing a number.
- `requirements-checker` treats a missing metric, threshold, or enforcement type on an AI/ML requirement as a **Blocker** — the same severity as a missing AC on any other requirement — and does *not* flag a well-formed eval-gated AC for lacking Given/When/Then.
- `epic-to-story-decomposer` carries the pattern down to story level, inheriting the requirement's threshold where it applies at story scope, or stating a story-specific one when the requirement's threshold is an aggregate the story only partially contributes to.
- `story-checker` applies the equivalent quality bar (missing metric / vague threshold / missing enforcement type = a finding) and flags the inverse case — a non-deterministic story written with only Given/When/Then and no eval fields at all.

**Assumption this pattern relies on:** whoever is operating an LLM through this pipeline can correctly identify when a requirement or story describes non-deterministic behavior. The skills define it as "won't return the exact same result twice given the same input" — model outputs, classifications, generated text, rankings, recommendations. When genuinely unsure, the safer default is to ask, not to silently pick Given/When/Then for something that's actually non-deterministic.

---

### 5.10 `adr-writer` — Stage 9 (runs once per decision)

**Purpose.** Writes Architecture Decision Records — a structured record of a genuine architectural fork, the real alternatives considered, the trade-offs, the decision, and its consequences, traced back to the requirements/NFRs/epics that drove it.

**Trigger phrases.** "write an ADR," "document this decision," "record why we chose X," "should we go with A or B."

**Inputs.** `{slug}-epics.md` and `{slug}-nfrs.md`. If NFRs aren't supplied, the skill can still write an ADR but flags that decision drivers may be incomplete.

**Outputs.** One file per decision: `{slug}-adr-<ADR-ID>-<short-name>.md` — Context, Decision Drivers (cited `REQ-NFR-*`/`EPIC-*` IDs), Options Considered, Decision, Consequences (Positive and Negative — honestly, not as marketing copy), Related Decisions, and (Phase 1 addition) an **Agent-Readable Summary**.

**Core process.** Identify the decision precisely → list genuinely viable options (never pad with a straw-man alternative) → evaluate against cited constraints, not abstract best practice → state the decision and the deciding factor plainly → state consequences honestly, including what gets harder → assign a permanent sequential `ADR-XXX` ID (superseding creates a new ADR, never edits the old one) → cite the specific drivers → **write the Agent-Readable Summary last**, once everything else is settled.

**Key rule.** If, honestly, only one option was ever viable, say so plainly rather than inventing a rigged comparison — a rigged ADR is worse than no ADR.

**What changed and why (Phase 1) — Agent-Readable Summary.** A one- or two-sentence, self-contained, imperative field naming both what was chosen **and what was explicitly rejected** — e.g. *"Use Apache Kafka for all event streaming; do not introduce RabbitMQ, SQS, or another broker — a second broker technology was explicitly rejected for operational overhead."* This exists because a coding agent reading a large repo will happily re-propose a rejected alternative if the only record of the rejection is buried in prose several paragraphs down; a summary that names the choice without naming the rejection doesn't actually stop the re-litigation this field exists to prevent. **Assumption:** any coding agent or engineer working from this repo reads ADR frontmatter/summaries before proposing a technology choice that an ADR already covers — the field only has value if it's actually consulted.

**Downstream consumer.** `solution-architecture-writer` treats every ADR with `status: accepted` as binding, not a suggestion.

---

### 5.11 `solution-architecture-writer` — Stage 10

**Purpose.** Produces a high-level solution architecture (component breakdown, diagram, NFR mapping, integration/data-flow description) from epics, NFRs, and accepted ADRs. High-level design only — not detailed schemas, class diagrams, or full API specs.

**Trigger phrases.** "design the architecture," "write an HLD," "what's the architecture for this," "draw the architecture."

**Inputs.** `{slug}-epics.md` and `{slug}-nfrs.md` (both required for a design worth checking later — if NFRs are missing, the skill still produces the architecture but flags that it can't be validated against non-functional constraints), plus any `{slug}-adr-*.md` files. `status: accepted` ADRs are binding; `proposed`/`rejected` ones are context only.

**Outputs.** `{slug}-architecture.md` — Component Overview (`COMP-XXX`, with Satisfies column), a Mermaid architecture diagram, per-component detail (interfaces, data owned, dependencies), NFR Mapping table, Data Flow & Integration narrative, ADRs Applied, (Phase 1 additions) **Pre-Mortem** and **Downstream Contract**, and Open Questions/Risks.

**Core process.** Read epics, NFRs, and every ADR before designing anything → identify components around business responsibility, not technology → for each component: responsibility, interfaces, data owned, `EPIC-*`/`REQ-*` traceability → describe and diagram integration/data flow (Mermaid, mandatory) → map every NFR to the component(s) responsible, with a "how" note → surface anything that doesn't map cleanly rather than forcing a bad fit → assign permanent sequential `COMP-XXX` IDs → **run a pre-mortem before finishing**.

**What changed and why (Phase 1) — Pre-Mortem.** A distinct, deliberately adversarial pass, separate from Open Questions/Risks: assume the design has already shipped and failed badly in production six months out, then work backward to the two or three most plausible reasons why. Typical angles: a component boundary that forces cross-team coordination every release; an NFR mapping that works at expected load but not 5x; a silent single point of failure; a wrong assumption about an external system. Open Questions captures what's noticed while drafting; the pre-mortem exists specifically to surface what prose-drafting doesn't.

**What changed and why (Phase 1) — Downstream Contract.** A short table at the top of the document stating what each consuming stage reads from this file and what it's expected to give back — making the handoff explicit in the document itself, not just scattered across other skills' pipeline-contract tables.

**Common failure modes.** Naming components after technology/layers instead of responsibility; leaving an NFR unmapped; contradicting an accepted ADR without calling it out; over-specifying implementation detail that belongs in a later design phase; skipping the pre-mortem or writing it as a restatement of Open Questions.

**Downstream consumer.** `architecture-vs-requirements-checker`.

---

### 5.12 `architecture-vs-requirements-checker` — Stage 11 [GATE — final numbered stage]

**Purpose.** Cross-checks a solution architecture against its source requirements and NFRs — not just presence/absence of coverage, but whether the design as described is *plausible* for meeting stated NFR targets.

**Trigger phrases.** "validate this architecture," "check this against requirements," "does this design actually meet our NFRs," "is this ready to build from."

**Inputs.** Three required files: `{slug}-requirements.md`, `{slug}-nfrs.md`, `{slug}-architecture.md`, with matching `initiative`. If NFRs are missing, functional coverage is still checked, but NFR conformance/plausibility is explicitly marked unverified — never silently treated as passing.

**Outputs.** `{slug}-architecture-review.md` — Functional Coverage Matrix, NFR Coverage & Plausibility table, Gaps (with severity), Orphan Components, ADR Conformance, Verdict, `review_verdict`: `architecture_sound` | `gaps_found` | `significant_gaps`.

**Core process.** Functional coverage (every `REQ-BR-*`/`REQ-FR-*` maps to an owning component, ID by ID) → NFR coverage (every `REQ-NFR-*` maps to a component with a "how") → **NFR plausibility** (read the actual described path and sanity-check whether the design could plausibly hit the target — not just whether the target is mentioned) → orphan components (unexplained scope vs. legitimate infra necessity) → ADR conformance → component boundary sanity (vague responsibilities that two engineers would build differently).

**Plausibility Assessment values:** **plausible** (design credibly supports the target) / **questionable** (target may be achievable but an expected mechanism is missing — flag for architect confirmation) / **implausible** (design and target are hard to reconcile as written).

**What changed and why (Phase 1) — Cost plausibility.** Extended the plausibility discipline, previously applied to latency/availability/throughput, to the new Cost & Resource Ceilings NFR category (5.4). A per-request LLM call in a hot path with no cache, no batching, no cheaper-model fallback, and a stated cost ceiling is implausible on its face — nothing in the design explains how the ceiling is actually enforced. A cost NFR with no component even attempting to control it is a straightforward coverage gap; where a control exists but its adequacy is uncertain (e.g., a cache is mentioned but its expected hit rate isn't), it's a questionable/plausibility judgment call, treated the same as any other NFR family.

**Routing.** Routes back to `solution-architecture-writer` if not `architecture_sound`. This is the last numbered stage — there is no stage 12. Once clean, the initiative's design is validated and ready to build from, and `traceability-matrix-generator` can produce a full end-to-end view.

---

### 5.13 `traceability-matrix-generator` — Utility (runs at any point)

**Purpose.** Aggregates whichever pipeline documents currently exist into a single end-to-end traceability matrix, one row per lowest-level traceable item, from stakeholder need through architecture component — flagging every broken or missing link explicitly rather than leaving cells blank.

**Trigger phrases.** "traceability matrix," "RTM," "show me the full chain," "can we trace this end to end," "what maps to what here."

**Inputs.** Whichever of the pipeline's files exist for a given `{slug}` — works with a partial chain; doesn't require the full 11 stages to be complete.

**Outputs.** `{slug}-traceability-matrix.md` — the matrix itself (columns present reflect which stages had a source file), Broken/Missing Links, Coverage Statistics (computed only for stages actually represented), Files Not Yet Available.

**Key rule — never fabricate a link.** If a citation connecting two IDs isn't actually present in the source text, mark it unlinked and let the finding stand, even if the connection seems obvious from naming similarity. This is an aggregator and mirror, not an editor — it never fixes or improves the underlying documents, only reports on them.

**Assumption.** Row granularity follows whichever level of decomposition is finest and actually available (story if stories exist for that epic, else the epic itself, else the requirement, else the need) — it does not pad the matrix with redundant rows at multiple levels for the same item.

---

### 5.14 `domain-glossary-builder` — Utility (runs at any point)

**Purpose.** Scans whichever pipeline documents exist and builds/updates a canonical glossary of domain terms, explicitly flagging terminology drift — the same concept named differently across documents written in separate skill runs (e.g., "customer" in requirements, "client" in an epic, "account holder" in the architecture).

**Trigger phrases.** "build a glossary," "define our terms," "are we being consistent with terminology," "domain model," "data dictionary."

**Inputs.** Whichever pipeline files are available — useful with as few as one document, more useful with more.

**Outputs.** `{slug}-glossary.md` — canonical term + definition (grounded in actual document usage, not a generic dictionary definition), aliases/synonyms with citing documents, explicit drift findings, Open Questions for genuinely undecided terms.

**Key rule.** Read-only auditor — flags drift for a human or a re-run of the originating skill to reconcile; never silently rewrites a source document. Every glossary entry must be traceable to real usage in real source material — no padding with terms that "should" be there but aren't actually used.

---

## 6. How an LLM should operate this pipeline

Given a request, determine the correct entry point using this decision procedure:

1. **Is there raw, unstructured discovery material** (notes, a transcript, a rough ask) **and no requirements document yet?** → Start at `stakeholder-discovery` (5.1), unless the material is already clear and pre-digested, in which case `requirements-writer` (5.2) may be invoked directly on the raw input.
2. **Does a `doc_type: requirements` file already exist, and the request is to review/validate it?** → `requirements-checker` (5.3).
3. **Does a `doc_type: requirements` file exist, and NFR coverage is thin or the request explicitly asks for NFRs?** → `nfr-elicitor` (5.4).
4. **Is the request to organize requirements into delivery-sized groupings** ("epics," "backlog," "workstreams")? → `requirements-to-epics` (5.5), then optionally `epics-vs-requirements-checker` (5.6).
5. **Is the request to turn an epic into sprint-ready tickets?** → `epic-to-story-decomposer` (5.7), then optionally `story-checker` (5.8).
6. **Is the request to record or challenge a specific technical choice?** → `adr-writer` (5.10).
7. **Is the request to design the system, or produce an HLD?** → `solution-architecture-writer` (5.11), then `architecture-vs-requirements-checker` (5.12).
8. **Is the request for a full-chain view or a consistency/terminology check, at any point in the above?** → `traceability-matrix-generator` (5.13) or `domain-glossary-builder` (5.14) respectively — these never block or gate the numbered chain.

At every gate stage (3, 6, 8, 11), **do not proceed downstream automatically** on a non-passing `review_verdict` without either fixing the flagged issues (by re-running the authoring stage) or an explicit human decision to proceed anyway with known gaps. Silently continuing past a failed gate defeats the purpose of having one.

When a request doesn't specify which stage to run, prefer the **earliest stage for which the required input doesn't yet exist** — running a downstream skill against a missing or unreviewed upstream artifact produces output that inherits and compounds the upstream gap.

## 7. Change log — Phase 1 (this pack)

**8 skills patched.** Full before/after rationale is in each skill's subsection in Section 5:

| Skill | Change |
|---|---|
| `requirements-writer` | Eval-gated AC (metric + threshold + Gate/Monitor) for AI/ML requirements |
| `nfr-elicitor` | Added Cost & Resource Ceilings as a 13th NFR category |
| `adr-writer` | Added Agent-Readable Summary field |
| `solution-architecture-writer` | Added Pre-Mortem section + Downstream Contract table |
| `requirements-checker` | Recognizes the eval-gated AC shape instead of penalizing it |
| `epic-to-story-decomposer` | Carries eval-gated AC down to story level |
| `story-checker` | Recognizes the eval-gated AC shape at the story-review gate |
| `architecture-vs-requirements-checker` | Extends NFR plausibility checking to Cost |

**5 skills unchanged this round**, included in this pack for completeness: `stakeholder-discovery`, `requirements-to-epics`, `epics-vs-requirements-checker`, `traceability-matrix-generator`, `domain-glossary-builder`.

**Known open item, not yet addressed:** `stakeholder-discovery` has no equivalent to an early value/feasibility/viability risk gate before an initiative proceeds to requirements — identified as a genuine gap against the AI Factory reference material this pack was benchmarked against, but scoped for a later phase (a validation-loop-planner concept) rather than folded into this skill, to avoid blurring "capture needs" with "decide whether to proceed" in one skill.

## 8. Verification checklist after applying this pack

- [ ] Run `requirements-writer` on a feature with at least one AI/ML-describable requirement; confirm the AI/ML? column and the metric/threshold/enforcement AC appear.
- [ ] Run `requirements-checker` against that output; confirm it passes cleanly (does not flag the eval-gated AC for lacking Given/When/Then) and correctly blocks on an AI/ML requirement missing any of the three fields.
- [ ] Run the chain through `requirements-to-epics` → `epic-to-story-decomposer`; confirm the eval-gated AC shape survives into the story file rather than being flattened into Given/When/Then.
- [ ] Run `story-checker` against that story file; confirm it accepts the eval-gated shape as Testable.
- [ ] Write an NFR set including a Cost & Resource Ceilings entry; run `solution-architecture-writer` then `architecture-vs-requirements-checker`; confirm the cost NFR gets a plausibility assessment, not just a presence/absence check.
- [ ] Write an ADR; confirm the Agent-Readable Summary names both the chosen option and what was explicitly rejected.
