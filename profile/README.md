# Context Continuum

> Operating discipline for multi-agent production systems.
> The public face of a two-person vector-management-tech LLC.

We're a two-person operation — **Randall** and **Owen** — running a
federated, multi-location, multi-machine Claude-agent environment in
production. The agents write the code. We write the operating system
that keeps them productive across locations, sessions, and compaction
events: the doctrines, the substrate-boundary checks, the federation
plumbing, the recovery patterns when something deadlocks at 3am.

This org is where that operating discipline gets distilled into a
form other people can read.

## What this is

Three repositories, all prose, no production code:

- **[operating-handbook](https://github.com/Context-Continuum/operating-handbook)**
  — the playbook for running multi-agent systems without them eating
  themselves. Inverse-pyramid format, load-bearing items first.
  Covers session pre-flight, boot ordering, durable session handoffs,
  channel routing, identity federation across locations, and the two
  doctrines that catch most of what goes wrong:
  PROCESS FORTIFICATION and SUBSTRATE-BOUNDARY DISCIPLINE.

- **[case-studies](https://github.com/Context-Continuum/case-studies)**
  — worked examples of substrate-boundary thinking applied to real
  production bugs. Each case study walks the chain: symptom → root
  cause → substrate-side fix → regression test → durable receipt.
  These are receipts for the discipline.

- **(this repo)** — the front door. Profile, narrative, contact.

## What we actually run

A federated multi-cluster Claude-agent operation spanning two
physical locations:

- **Cluster A** — multiple Claude sessions on Randall's machines
  coordinating through a local scratchpad daemon
- **Cluster B** — Owen's machine(s) running their own daemon +
  local agents
- **Shared surface** — a Firestore-backed Mission Control feed, a
  Google Drive build-lore archive, and a structured cross-cluster
  channel that respects boundaries so each cluster's internal
  chatter doesn't pollute the other's wake-routing layer

Things this system has produced under our direction:

- A Mission Control progressive web app with cross-agent dispatch,
  live chat between clusters, share-target routing from phones, and
  PWA push delivery
- A scratchpad-to-Firestore-to-Drive bridge for durable cross-cluster
  context and an archival build-lore corpus
- A wake-reliability harness that classifies each session's
  breadcrumbing as fired-on-time / late / early / missed / never-
  reached, and reconciles nightly
- A rule-based curator that turns scratchpad activity into queryable
  bulletpoints with zero LLM tokens consumed (deterministic, audit-
  able, free)
- A typed-trio observation substrate (claim/evidence pairs with
  refuse-at-write enums) for auditable cross-agent state
- An embedded vector database (hard fork of Qdrant) shaped into a
  sellable library product line

The interesting part isn't the code — agents wrote most of it. The
interesting part is what gets crystallized when we run that system
hard enough to find the bugs:

- **Refuse-at-write boundaries.** Bad data never lands in the
  substrate in the first place, so downstream consumers don't need
  defensive re-validation.
- **Substrate-not-proxy measurement.** When we need to know
  something, we read the authoritative source — never estimate from
  a side channel.
- **Process fortification.** If a discipline depends on an agent
  *remembering* to check, we fortify it with a deterministic signal.
  Agents reliably respond; they unreliably introspect.
- **Federated channel routing.** Each cluster's daemon is local;
  cross-cluster traffic goes through Firestore. Wake routing
  respects cluster boundaries so neither cluster spams the other.
- **Breadcrumb-at-threshold session handoffs.** At a measured
  context fill (~88%), the session writes a forward-looking
  breadcrumb before degradation. Continuity survives compaction.

These show up across the handbook and the case studies as concrete
working patterns, not abstractions.

## Selected receipts

- 6 substrate findings (F1–F6) surfaced in a single ~24h dogfood
  session, all closed at the right architectural boundary, all with
  regression tests
- A multi-doctrine operating handbook (~10k lines internal,
  sanitized public V0 here) refined across months of multi-cluster
  production
- A federated agent operation that has coordinated across two
  locations through multiple long sessions without deadlock,
  wake-storm, or cross-cluster impersonation incident
- Hook telemetry sink with schema validation + queryable read API
  for every fire/suppress/error of every cluster discipline hook

## Stack we operate

Claude Code as primary IDE, Anthropic API, Firebase (Firestore +
Cloud Functions + Hosting + Cloud Messaging), Google Drive API via
Domain-Wide Delegation, Node 20, Python 3.12, PowerShell on Windows
plus zsh/bash on macOS, an embedded vector database (PhaseShift
Engine, Rust 2024), Firebase Auth + Google Workspace identity model
for federated multi-operator attribution.

## Contact

**Randall** — [email or LinkedIn placeholder]
**Owen** — [email or LinkedIn placeholder]

For inquiries about the operating discipline, the handbook, or
about retaining either of us for collaborative-agent infrastructure
work, either contact works.

---

*Context continuity across compaction, across sessions, across
agents, across clusters, across locations. That's the work.*
