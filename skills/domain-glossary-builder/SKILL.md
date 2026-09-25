---
name: domain-glossary-builder
description: Scans whichever of a pipeline's documents currently exist and builds (or updates) a canonical glossary of domain terms and entities, flagging terminology drift — the same concept referred to by different names across different documents ("customer" in requirements.md, "client" in the architecture doc, "user" in the stories) — so the whole chain converges on one vocabulary. Trigger on "build a glossary," "define our terms," "are we being consistent with terminology," "domain model," "what do we call this," "data dictionary," or whenever multiple pipeline documents exist and terminology consistency matters — which is most of the time in a multi-stage chain written across separate skill runs. This is a read-only auditor — it flags terminology drift for a human (or a later run of the producing skill) to reconcile; it does not silently rewrite any of the source documents itself.
---

# Domain Glossary Builder

## Pipeline contract

This is a utility skill for an 11-stage BA-to-architecture pipeline. Every stage reads and writes plain Markdown **files** carrying a small YAML frontmatter block, so one stage's output is literally the next stage's input file.

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
| 11 | architecture-vs-requirements-checker | `{slug}-requirements.md` + `{slug}-nfrs.md` + `{slug}-architecture.md` | `{slug}-architecture-review.md` (`doc_type: architecture_review`) |

This skill isn't a numbered stage — it can run at any point once at least two or three of the files above exist, and is worth re-running any time a new pipeline document is added, since that's exactly when new terminology tends to get introduced. `{slug}` is the kebab-case initiative identifier carried in every document's frontmatter and reused across the whole chain — it's how you find all the files belonging to one initiative.

**Input contract for this skill:** reads whichever of the pipeline's files are available for the given `{slug}`. If it's unclear which files belong to the initiative or which ones the user wants included, ask; otherwise work with whatever's supplied, even if it's just one or two documents. More documents make the drift-detection more useful, but this skill has a real job to do with as few as one.

**Output contract for this skill:** always write your result to an actual `.md` file named `{slug}-glossary.md`, with the frontmatter block shown in Output Format below. Tell the user the filename when done, and if terminology drift was found, name specifically which documents (and, where relevant, which producing skill) should be re-run or hand-edited to converge on the canonical term.

## Role

Each stage in this pipeline is a fresh skill run — it doesn't carry forward a living memory of exactly which noun an earlier stage used for a given actor or entity. That's how terminology drift creeps in: `requirements-writer` calls someone "the customer," a later `epic-to-story-decomposer` run (working from an epic that paraphrased the same idea) calls the same person "the user," and by the time `solution-architecture-writer` describes a data model it's "the account holder." Each individual choice is reasonable in isolation. The problem is that the drift compounds silently across a long chain — by stage 9 or 10, a reviewer skimming the architecture doc may not even realize "account holder" and "customer" are meant to refer to the same person, and that ambiguity can leak into a data model, an API contract, or a story's acceptance criteria as if they were two different things.

Your job is to make that drift visible, not to fix it. You read across whatever documents exist for an initiative, build a canonical glossary grounded in how the documents actually use each term, and call out every place the same concept goes by more than one name. Reconciling it — picking the one true term and updating the documents that used something else — is a decision for a human, or for a re-run of whichever skill produced the inconsistent document. Don't silently rewrite any source file yourself.

## What to build

**Extract candidate terms.** Pull recurring nouns that matter to the domain: actors/roles (who does things), key entities/data objects (what the system tracks), business process names, and system/component names. Skip generic words that aren't domain-specific — "request," "system," "data" on their own rarely deserve a glossary entry unless this particular domain has given them a specific, narrower meaning.

**Write a canonical definition for each term**, grounded in how the documents actually use it — not a generic dictionary definition that happens to sound right. If the requirements doc defines a "claim" a certain way, that's the definition; don't substitute an insurance-industry textbook definition that might not match this initiative's actual usage.

**Detect aliases and synonyms.** Scan across documents for likely alternate names for the same concept: similar context, similar role in a sentence, or an explicit "aka"/"also known as" mention. List every alias against its canonical term, and cite which document(s) used which variant.

**Flag terminology drift as a finding.** Don't just quietly pick a canonical term and move on — that hides the problem instead of solving it. Call out specifically where the drift occurs: which documents used which term, and what it would take to converge.

**Flag genuinely ambiguous terms as Open Questions.** Some terms are used inconsistently enough, or vaguely enough, that the real issue isn't a naming choice — it's that nobody has actually decided what the term means. Route those to Open Questions rather than guessing at a definition.

**Don't pad the glossary.** Every entry should be traceable to real usage in a real document. If a term isn't actually used anywhere in the source material, it doesn't belong here even if it feels like the kind of term a glossary "should" have.

## Output format

Produce a single Markdown **file** — frontmatter plus a Confluence-style body:

```markdown
---
doc_type: glossary
schema_version: 1
initiative: <slug>
generated_by: domain-glossary-builder
source_files: [<list every file actually read>]
date: <today, ISO format>
---

# [Initiative Name] — Domain Glossary

## Summary

[2-4 sentences: how many terms, whether terminology drift was found and how significant it looks, and the overall picture of vocabulary consistency across the documents scanned.]

## Glossary

| Term | Definition | Aliases Found | Documents Using It |
|---|---|---|---|
| Account Holder | ... | "Customer" (requirements.md), "Client" (architecture.md) | stakeholder-needs.md, requirements.md, architecture.md |
| Claim | ... | — | requirements.md, epics.md |

## Terminology Drift

| Concept | Variants Found | Where | Recommendation |
|---|---|---|---|
| The person who owns an account | "Customer", "Client", "Account Holder", "User" | requirements.md uses "Customer"; architecture.md uses "Account Holder" and "Client" interchangeably; epic-003-stories.md uses "User" | Standardize on "Account Holder" (most precise; distinguishes from other user types like "Agent") — re-run or hand-edit requirements.md and the affected stories file |

[Omit this section, or state plainly that none was found, if the documents scanned are already consistent.]

## Open Questions

[Terms used inconsistently or vaguely enough that their actual meaning needs a business decision, not an inferred definition. Omit if none.]

## Documents Scanned

[List every file read, with its doc_type, so it's clear what this glossary does and doesn't cover.]
```

Save the file as `{slug}-glossary.md` and tell the user the filename, plus — if drift was found — which specific documents or producing skills to revisit to converge on the canonical terms.

## Calibration notes

Don't manufacture drift findings where the documents are actually already consistent. A clean glossary with an empty or absent Terminology Drift section is a good outcome, not a sign you didn't look hard enough — real BA teams that write carefully do sometimes stay consistent, and inventing tension where none exists just trains people to distrust the report.

Ground every definition in actual usage from the source documents, not a textbook or industry-standard definition that might not match how this specific initiative uses the term. Two initiatives can both use the word "claim" and mean genuinely different things by it.

If only one document exists so far, still produce a glossary — there's nothing to cross-check for drift yet, so say that plainly in the Summary rather than forcing a Terminology Drift section to appear. Note that the glossary will get more useful as more pipeline documents accumulate, and is worth re-running once they do.
