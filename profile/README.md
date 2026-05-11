# Context Continuum

> Operating discipline for multi-agent production systems.

We're a small team running heterogeneous-runtime AI agent clusters
in production. The agents write the code. We build and operate the
substrate that keeps them productive — across machines, runtimes,
clusters, and compaction events.

## What we actually run

Our primary cluster: **five agents on two machines**, mixed runtime
— three Claudes (`Mac/Claude-A`, `Mac/Claude-B`, `Win/Claude`) and
two Geminis (`Mac/Gemini`, `Win/Gemini`), per the agent registry at
`src/agent_identity.js:27-67`. The Geminis operate in two modes:
upstream as research-and-planning agents (formulate plans, post to
the cluster, the Claude team iterates) and downstream as autonomous
teammates handling smoke tests, benches, and audits for the Claude
team. Each machine runs its own daemon + bridge pair — the
**Voltron pattern**: every node capable alone, more powerful
federated.

Across clusters, **a different memory engine on each side**. Our
cluster runs PhaseShift Engine (more on that below); the peer
cluster runs a different vector backend entirely. The federation
protocol — scratchpad daemon, Firestore-mediated channel, ambassador-
pattern agent representation — is engine-agnostic *by demonstration*,
not by claim. We've proved interop by running different stacks, not
by adopting the same one.

Cross-cluster work happens via an **ambassador agent**
(`GoldenGoose/Claude`, in `ClusterInterface.jsx`'s CLUSTER_TREE) —
one named agent from the peer cluster, seconded into ours for
projects that need representation on both ends. The mechanism is
proven end-to-end; we use it when projects span, not as a daily
routine.

## The substrate

**PhaseShift Engine** — Rust 2024, our agentic memory layer. Recall
scoring is `cosine_similarity × max(importance, importance_floor) ×
decay`, with `decay = 0.5 ^ ((now − last_accessed) / half_life)` and
`half_life = base × (1 + ln(1 + access_count))` for sub-linear
spaced-repetition behavior (`docs/MEMORY.md:40-49`). Half-life
anchored on `last_accessed` not creation timestamp, so frequently-
recalled memories stay strong as the underlying text ages. Air-
gapped local-first design, P2P sync across machines, MCP endpoint
for direct agent integration.

**Trio observation substrate** — typed `(claim_kind, evidence_kind)`
pairs validated at the write boundary. Refuse-at-write rejection of
unknown enums + missing `authored_by` fields stops bad observations
from landing in the substrate (PR #79 layered with PR #81 —
SUBSTRATE-BOUNDARY DISCIPLINE applied at two layers of the same
write path). *The trio shape is informed by ACE's reference design;
the refuse-at-write enforcement is our substrate-boundary
contribution.*

**Pull-routing with substrate-honest reason tags** — agents claim
tasks matching their `capability_offer`. The router distinguishes
`claim_not_visible` (daemon read-after-write propagation lag) from
`lost_tiebreak` (actual competitor) instead of collapsing both into
"outcompeted" (F5 / PR #85). Surfacing the actual root cause
prevents agents from backing off pointlessly. *The pull-route +
capability-offer pattern is informed by ACE; the F5 substrate-
honesty distinction is our addition.*

**Rule-based curator** — turns scratchpad activity into queryable
bulletpoints with **zero LLM tokens consumed** (deterministic,
auditable, free). Built as the first product of one of our
substrate upgrades. *Some tool-call functionality threaded in from
LocalForge / Serena; the curator's design, the bulletpoint emission
pipeline, and the zero-LLM-tokens framing are ours.*

**Hook telemetry sink (B1)** — every fire/suppress/error of every
cluster discipline hook gets schema-validated and append-only
logged to `~/.claude/state/hook_telemetry.jsonl`. Read API via
`bridge.hook_telemetry_read.query_telemetry()`. **Correction events
use append-only join keys** (`corrects_session_id`,
`corrects_ts_ms`) — post-hoc patches surface alongside the
originals they corrected, without mutating source lines. Audit-trail
integrity preserved.

**Wake-reliability harness** — three-module nightly pipeline
(`wake_predictor.py` → `wake_observer.py` → `wake_reconciler.py`,
@02:00 local) that classifies each session's breadcrumbing into
five states: `fired_on_time` / `fired_late` / `fired_early` /
`missed` / `never_reached_threshold`. Monitor-fragility events that
were anecdotes are now substrate signals with recurring measurement.

**Breadcrumb @ 88% with substrate-not-proxy V2** — V1 used byte-
count estimation; rejected after measured calibration drift of 23%
between two same-cluster agents (`SOP_AUTOMATION_INVENTORY.md:31`).
V2 reads exact `usage.input_tokens + cache_creation_input_tokens +
cache_read_input_tokens` from `.jsonl` — the substrate-not-proxy
doctrine applied to context-fill measurement.

## Selected receipts

**Nine substrate findings (F1–F10, with F2 in two layers) closed
across a single 3h 44m session** — 2026-05-10 21:38 to 2026-05-11
01:23, verified against `git log --format="%aI %h"`. About 25
minutes per finding, each carrying root-cause analysis +
architectural-boundary decision + implementation + regression test +
inventory-row update + PR. Highlights:

- **F6** — wake-filter silently disabled cluster-wide on missing
  Firestore creds. Fixed at the wrapper boundary: auto-detect
  canonical install path + refuse-at-init substrate signal
  (stderr WARNING + B1 telemetry record) when both env and
  canonical-path resolution fail. See
  [F6 case study](https://github.com/Context-Continuum/case-studies)
  for full walkthrough.
- **F9** — long-running Monitor processes loading stale code into
  memory after PRs landed. Fixed: each Monitor records its own
  script's mtime at process start, stat's on every poll, and
  `os.execv`-restarts itself preserving argv when the mtime
  advances. Five acceptance tests pin the predicate.
- **F10** — Windows Task Scheduler spawning nightly crons in
  separate PowerShell processes that don't inherit `$env:*` set by
  the bridge launcher. Wrappers now auto-detect bridge creds from
  the canonical install path. Same pattern as F6 applied at the
  wrapper layer.

Plus the daemon-readiness wait we shipped today (2026-05-12) — the
bridge launcher now polls daemon `/healthz` at the wrapper boundary
before invoking python, so the boot-race that exited bridge on
first poll is closed.

## Stack we operate

Claude Code as primary IDE, Google Antigravity for Geminis,
Anthropic API, Firebase (Firestore + Cloud Functions + Hosting +
Cloud Messaging), Google Drive API via Domain-Wide Delegation,
Node 20, Python 3.12, PowerShell on Windows + zsh/bash on macOS,
PhaseShift Engine (our Rust 2024 memory engine), Firebase Auth +
Google Workspace identity model for federated multi-operator
attribution.

## Contact

[Inquiries: contact path TBD]

For questions about the operating discipline, the handbook, or
about retaining us for collaborative-agent infrastructure work.

---

*Context continuity across compaction, sessions, agents, runtimes,
and clusters. That's the work.*
