# Context Continuum

> Operating discipline for multi-agent production systems.

I direct teams of AI agents to build and run production software. The
agents write the code. I write the operating system that keeps them
productive — the doctrines, the substrate-boundary checks, the
recovery patterns when something deadlocks at 3am. This org is where
that operating discipline gets distilled into a form other people
can read.

## What this is

Three repositories, all prose, no production code:

- **[operating-handbook](https://github.com/Context-Continuum/operating-handbook)**
  — the playbook for running multi-agent systems without them eating
  themselves. Inverse-pyramid format, load-bearing items first.
  Covers session pre-flight, boot ordering, durable session handoffs,
  channel routing, and the two doctrines that catch most of what
  goes wrong: PROCESS FORTIFICATION and SUBSTRATE-BOUNDARY DISCIPLINE.

- **[case-studies](https://github.com/Context-Continuum/case-studies)**
  — worked examples of substrate-boundary thinking applied to real
  production bugs. Each case study walks the chain: symptom → root
  cause → substrate-side fix → regression test → durable receipt.
  These are receipts for the discipline.

- **(this repo)** — the front door. Profile, narrative, contact.

## What I actually do

I run a three-agent Claude cluster across two physical machines.
Multi-hour sessions, coordinated lanes, real production artifacts.
Things the cluster builds and ships under my direction:

- A Mission Control PWA with cross-agent dispatch, live chat, share-target
  routing, and PWA push delivery
- A scratchpad-to-Firestore-to-Drive bridge for durable cross-cluster
  context and build-lore archive
- A wake-reliability harness that classifies session breadcrumbing
  as fired-on-time / late / early / missed / never-reached
- A rule-based curator that turns scratchpad activity into queryable
  bulletpoints without any LLM tokens
- A typed-trio observation substrate (claim/evidence pairs with
  refuse-at-write enums) for auditable cross-agent state

The interesting part isn't the code — agents wrote most of it. The
interesting part is what gets crystallized when you run that system
hard enough to find the bugs:

- Refuse-at-write boundaries. Bad data never lands in the substrate
  in the first place, so downstream consumers don't need defensive
  re-validation.
- Substrate-not-proxy measurement. When you need to know something,
  read the authoritative source — don't estimate from a side channel.
- Process fortification. If a discipline depends on an agent
  *remembering* to check, fortify it with a deterministic signal.
  Agents reliably respond; they unreliably introspect.
- Breadcrumb-at-threshold session handoffs. At a measured context
  fill (~88%), the session writes a forward-looking breadcrumb
  before degradation. Continuity survives compaction.

These show up across the handbook and the case studies as concrete
working patterns, not abstractions.

## Selected receipts

- 6 substrate findings (F1–F6) surfaced in a single ~24h dogfood
  session, all closed at the right architectural boundary, all with
  regression tests
- A multi-doctrine operating handbook (~10k lines) refined across
  three months of multi-agent production
- A three-agent cluster that has coordinated through multiple
  same-cwd Voltron co-tenancy scenarios without deadlock or wake-storm
- Hook telemetry sink with schema validation + queryable read API
  for every fire/suppress/error of every cluster discipline hook

## Stack I operate against

Claude Code, Anthropic API, Firebase (Firestore + Functions + Hosting +
Cloud Messaging), Google Drive API via Domain-Wide Delegation, Node 20,
Python 3.12, PowerShell on Windows + zsh/bash on macOS, a hard-fork
embedded vector database (PhaseShift Engine, Rust 2024), Firebase
Auth + Google Workspace identity model.

## Contact

[Email or LinkedIn placeholder — fill in before this goes public-public]

---

*Context continuity across compaction, across sessions, across agents,
across clusters. That's the work.*
