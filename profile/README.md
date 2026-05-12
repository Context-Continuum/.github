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
protocol — local-daemon comms layer, Firestore-mediated channel,
ambassador-pattern agent representation — is engine-agnostic *by
demonstration*,
not by claim. We've proved interop by running different stacks, not
by adopting the same one.

Cross-cluster work happens via an **ambassador agent**
(`GoldenGoose/Claude`, in `ClusterInterface.jsx`'s CLUSTER_TREE) —
one named agent from the peer cluster, seconded into ours for
projects that need representation on both ends. The mechanism is
proven end-to-end; we use it when projects span, not as a daily
routine.

## Quickstart

The engine ships with a 15-minute "build a memory-aware agent from
cold daemon to first decaying recall" walkthrough at
`docs/QUICKSTART.md`. Path: start the daemon → `store_memory` a
fact → `recall` it via semantic search → re-recall and watch the
decay math run live → `delete_memory` to demonstrate the operations
surface. Single-binary, no cloud calls, no external API keys to
get to the first useful recall.

For substrate-level pre-flight,
[Context-Continuum/swarm's `smoke.py`](https://github.com/Context-Continuum/swarm/blob/main/smoke.py)
exercises every coordination primitive end-to-end against a running
daemon in seconds (register → claim → heartbeat → GPU mutex
round-trip → update → hive snapshot → negative session-token gate).
No Ollama or web access required.

## The substrate

**PhaseShift Engine** — Rust 2024, our agentic memory layer. Recall
scoring is `cosine_similarity × max(importance, importance_floor) ×
decay`, with `decay = 0.5 ^ ((now − last_accessed) / half_life)` and
`half_life = base × (1 + ln(1 + access_count))` for sub-linear
spaced-repetition behavior (`docs/MEMORY.md:40-49`). Half-life
anchored on `last_accessed` not creation timestamp, so frequently-
recalled memories stay strong as the underlying text ages. Air-
gapped local-first design, P2P sync across machines, MCP endpoint
for direct agent integration. Forked from Qdrant 1.17.1 — we
inherit a mature HNSW indexing backend and strip the enterprise
multi-tenancy surface in favor of agentic primitives. **Multi-
client by default**: writes serialized internally, reads
parallelized, per-call collection isolation — N agents share one
daemon without WAL-lock contention. **Two deployment modes from
the same binary**: long-lived MCP-over-HTTP daemon (multi-client,
multi-collection routing) or spawn-on-demand stdio subprocess
(simplest for stdio-only clients like Claude Desktop) — choose per
workload, no rebuild. **Tunable `MemoryConfig`**: `base_half_life_secs`,
`touch_on_recall`, `overfetch_factor`, `importance_floor`, and
more — same engine, different decay curves for long-context
research vs short-term operator chat vs deep-archive forensics.
**Auth-by-default on LAN bindings**: the daemon refuses to start
non-loopback without `PHASESHIFT_AUTH_TOKEN`, so it can't be left
accidentally exposed.

**1M-record validation, Tier 1 across the board**
(`docs/RELEASE_NOTES_v0.1.1.md`). 1M-record corpus, BGE-base 768d,
dense recall: **2h 53m wall-clock** end-to-end ingest, **26.51 ms
dense p50** (vs. v0.1.0's 720 ms brute-force scan — **27× lift**),
**15.41 ms LAN single-client p50** on a persistent HTTP connection,
ingest rate **stays flat** across 10k → 1M (100.6 → 96.21 rec/s, a
**5.5× lift** in sustained throughput vs. v0.1.0). The flat curve
is the load-bearing observation — 100× more records, dense p50
essentially unchanged. HNSW's log-N scaling validated empirically
rather than asserted. The first 1M run was a 16-hour ingest with
720-919 ms recall p50 — we'd shipped HNSW indexing as manual-only
so cold-ingest never triggered it, and recalls were brute-force
scans against a Plain segment. v0.1.1's `bulk_ingest` MCP tool +
automatic HNSW build closed the gap.

**ONNX embedder** — `crates/embed` ships an `OnnxEmbedder` (feature
`onnx`) on the `ort` 2.0 binding to ONNX Runtime 1.24, with
`load-dynamic` linking so the runtime DLL stays out of our binary
(user supplies it at runtime — single-binary positioning preserved).
Production model: **BGE-base-en-v1.5** (768d, BERT WordPiece,
512-token max sequence). A `HashEmbedder` fallback keeps the default
build dependency-light for callers who bring their own vectors.

**GPU acceleration, cross-platform** — the same ONNX export runs on
five execution providers: **Apple Neural Engine** via CoreML (Mac),
**NVIDIA CUDA** (Linux + Windows, with CUDA Graph + IoBinding for
sub-call overhead), **DirectML** (Windows AMD / Intel iGPU), **ROCm**
(Linux AMD), and CPU fallback. Build feature selects the EP set
(`cargo build --features coreml | cuda | directml | rocm`); runtime
`--ep` flag prioritizes (`--ep cuda --ep cpu`). The two measured at
production scale — **CoreML on Apple M3 (ANE-dispatched) and CUDA on
RTX-class (CUDA Graph + IoBinding)** — produce **byte-identical
recall results** on the same ONNX export, validating zero precision
drift between execution providers
(`results/cluster_ab_10k_bge_base.md`). DirectML and ROCm have
documented build paths and first-boot validation; scale benchmarks
for those two are a known canon-block gap
(`gpu_phase_8_results`), not a missing capability.

**Multi-language AST extraction** — `crates/ast` parses four
languages via Rust-native crates with zero tree-sitter / libclang
FFI: Rust (`syn`), Python (`rustpython-parser`), JavaScript
(`oxc_parser` — the Oxc/Rolldown parser), and TypeScript (same
parser via `SourceType::ts()`). Feature-gated per-language so the
default build stays slim. `MemoryCollection::ingest_source_file`
(in `crates/memory/src/ast_ingest.rs`) chains extraction → per-item
embedding → stable-concept-id upsert
(`sha256(file_stem__kind__name)[..8]`), so re-ingesting an updated
source file UPSERTS the matching memories in place rather than
producing duplicates. Code-search index that updates as the codebase
changes, with no FFI build complexity in the dependency tree.

**Single coordination model, three scales** — the same primitives
work whether agents are on one machine, one cluster, or across
clusters. On a single machine: local Claude and local Gemini share
the daemon and coordinate without leaving the box. Within a cluster
spanning multiple machines: each box runs its own daemon + bridge
pair (the Voltron pattern), and agents on different machines
coordinate through the cluster's shared substrate. Across clusters
of independent operators: a Firestore-mediated channel + ambassador-
agent pattern lets foreign clusters request attention without ever
touching local internals. The mechanisms don't change at the
boundaries — same coordination model, three operational scales.

**Wake-routing audience filter** — every agent comm is routed by
addressee. The wake filter surfaces only messages directly addressed
to this agent OR group broadcasts flagged with action-or-higher
urgency; informational traffic stays on the channel without waking
anyone. Empirically: ~95% of cluster traffic is filtered out before
it reaches an agent. **Cross-cluster wake** is an opt-in V1.2
protocol: a foreign cluster marks a Mission Control item with a
wake-target field, our filter picks it up via Firestore polling, an
anti-double-wake check prevents loops. Silently backwards-compatible
when Firestore credentials aren't configured — the same agent code
runs in daemon-only mode without modification.

**UDP tripwire mesh** — every successful write on a daemon emits a
lightweight UDP broadcast. Peer daemons listen and trigger a
reactive sync pull on receive (`UdpTripwireEmitter` +
`UdpTripwireListener` + `MeshSyncTrigger`, wired in
`src/bin/phaseshift.rs:846-1049`). **Sub-5-second cluster-global
write visibility on LAN** — UDP latency in milliseconds plus the
pull RTT, no peer needing to poll on a tight schedule. A periodic
full-sync backstop catches anything UDP drops. Fast path is the
tripwire; correctness guarantee is the reconciler. **`groups.yaml`
strong-consistency startup gate**: each daemon computes SHA-256 of
its local cluster-config on boot, probes known peers, and refuses
to start on hash mismatch (operator-overridable for legitimate
divergence mid-rollout). Cluster-config drift catches at startup
instead of silently mid-run.

**Cross-encoder reranker** — optional rerank stage between the
bi-encoder retrieval and the final top-N (`OnnxReranker` +
`OnnxRerankerConfig` in `crates/embed`; attached via
`with_reranker`, queried via `recall_reranked`). Cost: ~20-40 ms
per query at K=20 candidates on CoreML or CUDA — well within the
Tier 1 latency budget. Targets the deep R@1 ceiling that the
bi-encoder retrieval plateaus at. ONNX-backed so the same five-
provider execution stack applies (Apple ANE, NVIDIA CUDA, DirectML,
ROCm, CPU).

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

**Living SOP automation inventory** — every cluster norm has a row
in `docs/SOP_AUTOMATION_INVENTORY.md` tracking its automation state
(`automated` / `partial` / `prose-only`), the measurable signal that
triggers it, and the hook path that fires the reminder. Eight-plus
SOPs already on the automated tier: breadcrumb-at-threshold,
turn-zero Monitor arm, catch-up digest visibility, urgency
discipline, post-merge live-smoke, cross-cluster routing checks,
hook telemetry sink, wake-reliability harness. The inventory is
the receipts page for PROCESS FORTIFICATION DOCTRINE — when an SOP
gains a measurable signal it moves up the ladder, when a prose-only
norm gets identified it gets a row.

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

**Rule-based curator** — turns agent comm activity into queryable
bulletpoints with **zero LLM tokens consumed** (deterministic,
auditable, free). Built as the first product of one of our
substrate upgrades. *Some tool-call functionality threaded in from
LocalForge / Serena; the curator's design, the bulletpoint emission
pipeline, and the zero-LLM-tokens framing are ours.*

## Where the borrowed primitives stop and the production substrate starts

> *We take an inch but we make a mile.*

ACE gives a reference design for trio observations and pull-routing.
LocalForge / Serena give tool-call primitives. Those upstream
projects give us a real head start on the right shapes. Taking
borrowed primitives into our specific multi-cluster production
environment is a different problem from what those projects were
built to solve — what we add is refuse-at-write boundaries on every
observation, every claim, every state transition; the substrate-
honest reason tags; the hook telemetry sink; the F-finding closures;
the operating discipline that keeps it all coordinated. Borrowed
primitives plus our substrate equals something we can run at scale.
The next section is the receipts.

## Selected receipts

The receipts the reviewers ask for: real production-fragility bugs
caught and closed, root-cause-analyzed + tested + documented + PR'd
— not toy demos.

**Nine substrate findings (F1–F10, F2 in two layers) closed in a
single 3h 44m session** — 2026-05-10 21:38 to 2026-05-11 01:23,
verified against `git log --format="%aI %h"`. ~25 minutes per
finding, each carrying root-cause analysis + architectural-boundary
decision + implementation + regression test + inventory-row update
+ PR. These weren't bugs we already knew about; they were silent
failure modes the cluster surfaced *because* the substrate hooks
were watching. Highlights:

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

These are the kinds of silent failure modes that surface only in
multi-machine, multi-runtime, long-running production — not in
single-machine demos. The same substrate hooks that surface them
are the ones described in the substrate-section paragraphs above.

## Production discipline

- **Test coverage at the unit + integration layer.** Per-crate
  breakdown maintained in `CLAUDE.md`: 27 engine tests (manager +
  edge config), 30 embed tests (31 with `--features onnx`), 25
  memory tests (decay math + payload + collection), 7 MCP unit + 8
  stdio integration tests, +10 HTTP integration tests on a live
  socket with `--features daemon`, 4 REST server integration tests.
- **Release notes locked, tier promotion ceremonial.** v0.1.0
  shipped with the measured 10k cluster-A/B (CoreML M3 + CUDA RTX,
  both lanes on the same ONNX export, byte-identical recall
  cross-EP). v0.1.1 was a tier-promotion release — locked release
  notes lead with *"Tier 1 across the board. 1M validation
  complete"* and carry a v0.1.0-vs-v0.1.1 differential table for
  every metric (27× lift on dense p50 at 1M, 5.5× ingest rate
  lift). Release ceremony is the gate, not the announcement.
- **API_REFERENCE as source-of-truth.** `docs/API_REFERENCE.md` is
  regenerated against a live daemon — the MCP tool definitions come
  from `tool_catalog_daemon()` in `crates/mcp/src/tools.rs`, the
  HTTP route table from `router()` in
  `crates/server/src/handlers.rs` + `crates/mcp/src/transport_http.rs`.
  If a tool or route changes, regen catches the drift.
- **Canonical smoke after every change.** `cargo run --example
  proof_of_life` is the engine's "does it still work" check — must
  print `All proof-of-life checks passed`, cosine top-1 must be
  1.0000 for the probe vector. Convention: run after any non-
  trivial engine change before opening the PR.

## Stack we operate

Claude Code as primary IDE, Google Antigravity for Geminis,
Anthropic API, Ollama with Qwen 2.5 7B locally on each machine
(tokenless local inference for audit/bench/scrape-class work),
Firebase (Firestore + Cloud Functions + Hosting + Cloud Messaging),
Google Drive API via Domain-Wide Delegation, Node 20, Python 3.12,
PowerShell on Windows + zsh/bash on macOS, PhaseShift Engine (our
Rust 2024 memory engine), Firebase Auth + Google Workspace identity
model for federated multi-operator attribution.

## Security & deployment

- **Air-gapped local-first.** No external API dependencies in the
  hot path. Models bundled with the binary, vectors and metadata
  in a single data directory, no cloud calls required for any read
  or write. Deployable on a laptop with no network.
- **Auth-by-default on LAN bindings.** The daemon refuses to start
  bound to a non-loopback address without `PHASESHIFT_AUTH_TOKEN`
  set — a misconfigured launch can't be silently world-readable.
  Loopback binds (`127.0.0.1:7878`) run no-auth for local-only
  workloads.
- **Single-binary deployment.** One Rust binary handles every
  deployment mode (stdio subprocess, MCP-over-HTTP daemon, REST
  server). No Docker required, no Python runtime, no JVM. The ONNX
  Runtime DLL is user-supplied via `load-dynamic` linking so the
  binary stays under your control.
- **Narrow `drive.file` OAuth scope.** Mission Control's Drive
  integration can only see or edit files it created itself or files
  the operator explicitly attaches via Picker. Cannot read the rest
  of the operator's Drive — disqualifies the OAuth-creep concern
  up front.

## Mission Control

The operator-facing surface for the cluster — web UI accessible
from any browser or phone, backed by Firestore plus a per-machine
Python bridge. It's what makes the cluster operable without
opening a terminal.

- **Google Workspace SSO.** Single sign-on propagates operator
  identity to every surface that needs it: push registration,
  Drive uploads, audit records.
- **Cross-cluster + operator-to-operator chat.** Mission Control's
  chat surface carries operator-to-agent on any foreign cluster,
  agent-to-operator back, and operator-to-operator threads in
  multi-operator setups. Operator-to-operator threads also serve
  as POV context that the cluster's agents can reference when SOPs
  call for it. Intra-cluster agent-to-agent comms run on the local
  network daemon and don't surface here — routing-by-audience keeps
  each channel scoped to what actually needs to see it.
- **Tasks with review gate.** Assigner / assignee / due-date /
  status flow; per-event notification preferences; review gate
  before terminal state (submitted → revisions-requested →
  approved).
- **Milestones + decision lanes.** Milestone items mark
  substantive deliveries on the home feed. Decision items capture
  architectural choices + alternatives considered + the resolution
  — durable references rather than buried-in-chat history.
- **Drive-archived build-lore.** Substantive work auto-generates a
  500–1500 word extended report uploaded to a Drive folder via
  delegated-domain authority; the home-feed item carries a one-
  click link so deep context is never more than one hop away.
  Narrow `drive.file` OAuth scope — Mission Control can only see
  or edit files it created itself or files the operator explicitly
  attaches via Picker. It can't read the rest of your Drive.
- **GitHub webhook → items.** Commits flow into the home feed
  automatically — verified live this session, three commits
  landed as items within seconds of push.
- **File + link sharing from any device.** Operators on mobile (or
  any browser) push files and links straight to agents running on
  their home or office machines — the agent picks them up in
  context without a separate hand-off step. Run an errand, hand
  the workstation agent a PDF + a URL, keep moving. Wire format
  ratified by foreign-cluster ambassador review (10 redlines, 6
  folded, 2 clarified, 2 parked) so the protocol carries across
  clusters. 100 MB cap with resumable upload for large files;
  failed uploads surface as struck-through tiles with error
  tooltips, not silent drops; schema-versioned for forward-compat;
  both Drive view URL and content URL attached so PDFs and SVGs
  embed inline.
- **Per-event FCM push.** Notifications to operator phones with
  per-event opt-in toggles and quiet-hours support. Data-only
  payload shape, service-worker-rendered — works on PWA-installed
  Chrome Android with custom title/body, not the OS-default
  fallback.
- **Agent dispatch from any surface.** Operators or agents trigger
  agent runs from Mission Control or from peer-agent comms; a per-
  machine watchdog turns those triggers into actual local agent
  execution. Same model whether the trigger comes from an operator
  on a phone or from another agent on the cluster — Mission Control
  isn't just a read-only feed, it's a remote-control surface.

## Autonomous research swarm

A separate coordination plane on top of PSE. Eight dedicated MCP
tools (`register_drone`, `claim_task`, `update_task`,
`heartbeat_task`, `deploy_task`, `hive_status`, `gpu_acquire`,
`gpu_release`) let N drone processes register with the daemon,
claim work from a shared queue, run a research pipeline (search →
scrape → extract → synthesize → ingest against local Qwen for
tokenless inference), and push bounded follow-up topics back to
the queue.

Drones coordinate through the vector store itself. Each drone
recalls sibling research before generating its own queries and is
pushed toward orthogonal angles ("Boid Flocking Matrix"). Follow-up
topics go through a dedup gate that skips candidates the swarm
already covers. A heartbeat sweeper reclaims tasks if a drone
crashes mid-work. A GPU mutex serializes Ollama access so concurrent
drones on the same machine run under contention without thrashing.

Verified end-to-end on a live 3-drone run: every primitive — atomic
claim, flocking entanglement with contextually-appropriate sibling
selection, GPU serialization, sweeper-reclaim, dedup gate making
real contextual decisions (block one drone's near-duplicate,
accept another's orthogonal angle) — fired correctly on a real
research topic. Works for any workload that decomposes into
independent claims: parallel web scraping, multi-source research,
distributed audits.

The swarm is its own coordination plane; daily Claude/Gemini lane
work runs on pull-routing + trio observation. The two share the
PSE daemon and the agent identity surface, but the dispatch and
lifecycle primitives are distinct. Code:
[Context-Continuum/swarm](https://github.com/Context-Continuum/swarm).

## Contact

[Inquiries: contact path TBD]

For questions about the operating discipline, the handbook, or
about retaining us for collaborative-agent infrastructure work.

---

*Context continuity across compaction, sessions, agents, runtimes,
and clusters. That's the work.*
