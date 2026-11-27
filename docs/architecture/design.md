# RAZStudio OS — Architecture Design

**Specification baseline:** 2026-11-27
**Project:** NixOS — personal AI coding, research and engineering server
**Status:** Approved architectural baseline; implementation and hardware compatibility require evidence.

This file is one part of a coordinated three-document specification. Read `Requirements.md`, `design.md` and `tasks.md` together. The three files collectively replace the previous single-file master to reduce context and parsing load for coding agents.

**Document roles**
- `Requirements.md`: what the system must do, scope, constraints, quality attributes and acceptance criteria.
- `design.md`: how the system is structured, technology choices, model/agent architecture, security and locked decisions.
- `tasks.md`: implementation sequence, dependencies, execution rules and evidence required to mark work complete.

**Authority rule:** Requirements define outcomes; Design defines the approved architecture; Tasks define implementation order. If a task appears to conflict with a requirement or locked design decision, stop and report the conflict. Do not silently reinterpret the specification. Changes to locked decisions require owner approval.

## 0. Authority, change control and how to use this document

This file is the **single source of truth** for NixOS. It consolidates product requirements, architecture, hardware assumptions, security rules, implementation phases, task order and acceptance tests. Do not maintain competing requirements/design/tasks documents that can drift apart. Supporting documents may be generated from this master later, but this master remains authoritative.

### 0.1 Normative language

- **MUST / SHALL:** mandatory requirement.
- **MUST NOT / SHALL NOT:** prohibited.
- **SHOULD:** recommended unless a documented reason justifies deviation.
- **MAY:** optional.
- **P0:** required for the first bootable/installable ISO.
- **P1:** required for the first useful installed-host foundation.
- **P2:** core AI coding/research platform.
- **P3:** advanced capabilities and future expansion.

### 0.2 Change control

1. Every material architecture change MUST be recorded in `DECISIONS` within this document, with date, reason, alternatives, impact and migration notes.
2. Implementation agents MUST NOT silently alter requirements, remove features, change security boundaries or promote a prototype into production.
3. A change to a locked decision requires owner approval.
4. The Git commit, `flake.lock`, build manifest and test evidence identify what was actually built. This document alone does not prove implementation or hardware compatibility.
5. An agent may propose edits to this master, but the owner approves changes to locked architecture and scope.

### 0.3 Engineering truth rule

A plausible answer is not verified correctness. Compilation is not proof of correctness. A model's benchmark is not a guarantee on this hardware. A test is passed only when its command, environment, result and relevant logs are recorded. Never invent test output, source evidence, model compatibility or hardware measurements.

---


# PART III — HARDWARE BASELINE AND CONSTRAINTS

## 5. Known machine profile

| Component | Current information | Engineering implication |
|---|---|---|
| Motherboard | MSI B450M MORTAR MAX | Confirm BIOS version, PCIe link state and storage occupancy |
| CPU | AMD Ryzen 5 2400G, 4C/8T, Zen+ Raven Ridge | Modest CPU inference and orchestration capacity; no assumed modern AMD power features |
| Integrated GPU | Radeon Vega 11 | Keep as display/diagnostic fallback; do not assume ROCm support |
| GPU 1 | NVIDIA GTX 1650, 4 GB | Primary candidate for local inference experiments; exact PCI ID and variant required |
| GPU 2 | NVIDIA Quadro P1000, 4 GB | Secondary candidate for small inference/embedding experiments; Pascal compatibility must be checked |
| RAM | 48 GB, 3×16 GB, reported 3000–3200 MT/s | Verify actual frequency, channel mode, timings and stability |
| PCI_E1 | Board supports Gen3; Ryzen with Radeon Vega Graphics may negotiate x8 | Verify actual negotiated width/speed |
| PCI_E4 | Board supports Gen2 x4 | Verify actual negotiated width/speed |
| Storage | Not yet inventoried | Never assume device names or partition layout |
| Network | NIC/Wi-Fi details not yet verified | Detect before selecting firmware/modules |

### 5.1 Hardware facts and cautions

- Two 4 GB GPUs have **two independent VRAM pools**. They do not automatically become a unified 8 GB GPU.
- PCIe lane allocation and M.2 occupancy can affect available slots. On this motherboard, PCI_E4 is unavailable when an SSD is installed in M2_2. Verify the actual board and occupancy.
- The GTX 1650 exact variant matters; do not infer performance or encoder capabilities from the product name alone.
- Quadro P1000 is Pascal generation. Driver, CUDA toolkit, kernel module and runtime compatibility must be explicitly tested against the selected NixOS kernel and pinned packages.
- Do not assume ROCm support for Vega 11 or Raven Ridge.
- Do not force `amd-pstate` without verified CPPC support. Detect the actual frequency driver and supported governors.
- Preserve CPU thermal protections and boost behaviour initially; do not stack multiple power-management daemons.
- CPU-only boot and inference fallback MUST remain possible.
- NVIDIA driver failure MUST NOT prevent the host from booting or administering the server.

### 5.2 Hardware discovery gate

Before physical installation or GPU workload configuration, record:

- `lspci -nnk` for both NVIDIA devices and integrated graphics;
- GPU PCI IDs, bound drivers, kernel module state and negotiated PCIe links;
- `nvidia-smi` output when available;
- `lsblk`, `blkid`, SMART and filesystem information;
- motherboard BIOS version and UEFI state;
- DIMM slot population, memory frequency and stability;
- NIC/Wi-Fi chipset and driver;
- M.2_1/M.2_2 occupancy and any PCI slot disablement.

No physical disk may be selected by a guessed `/dev/sdX` name. Use model, serial, capacity and explicit owner confirmation.

---


# PART IV — SYSTEM ARCHITECTURE

## 6. High-level architecture

The system is divided into independently deployable services with explicit interfaces:

1. **Host OS and Hardware Layer** — NixOS, systemd, networking, storage, drivers, resource control.
2. **API Gateway and Identity Layer** — client authentication, quotas, policy, audit and routing.
3. **Agent Runtime** — agent definitions, role profiles, model selection, tools and lifecycle.
4. **PAESE** — parallel worker execution, supervisor, integration and reviewer dispatch.
5. **PACE** — durable task context and recovery.
6. **DRKE** — on-demand and scheduled research acquisition, extraction, provenance and freshness.
7. **Knowledge/RAG Layer** — PostgreSQL full-text search, pgvector and project context retrieval.
8. **Skill Development & Governance Engine** — skill drafting, testing, review, approval, versioning and rollback.
9. **Workspace and Tool Execution Layer** — isolated worktrees, containers, builds, tests and artifact storage.
10. **Observability and Recovery Layer** — metrics, logs, backups, health checks and restore procedures.

Services communicate through authenticated internal APIs and durable records. Model outputs and tool results are treated as untrusted input until validated.

## 7. Recommended initial technology stack

| Concern | Initial choice | Rule |
|---|---|---|
| OS | NixOS stable with flakes | Pin exact revision; never build release from floating unstable |
| Service manager | systemd | Native units, cgroups, restart policies, resource limits |
| Main service language | TypeScript on supported Node.js LTS | Keep runtime count low |
| API | Fastify | Versioned schemas and authentication middleware |
| Database | PostgreSQL | Durable source of truth for tasks, audit, metadata |
| Vector retrieval | pgvector + PostgreSQL full-text search | No second vector DB until benchmarks prove need |
| Durable workflows | Evaluate Temporal | Use only if footprint/operations justify it; PostgreSQL-backed queue is fallback |
| Local inference | llama.cpp server initially | CPU fallback; CUDA only after GPU qualification |
| Hosted inference | Provider-neutral adapters | Credentials stay server-side |
| Web acquisition | Crawlee HTTP-first | Playwright only for pages that require browser execution |
| Extraction | Readability/Trafilatura-style extraction and structured parsers | Preserve raw source and extraction version |
| Repository acquisition | Git CLI and supported forge APIs | Pin commit and preserve repository provenance |
| Agent tools | Official MCP SDKs plus controlled internal tool registry | Least privilege and allowlists |
| Isolation | Podman and systemd/cgroups | No Kubernetes initially |
| Metrics | Prometheus/Grafana later | Add after core health/metrics contract exists |
| Search API | Provider adapters; evaluate Exa or alternatives | No single search vendor lock-in |
| Secrets | systemd credentials or suitable secret store | Never commit or log secrets |

### 7.1 Complexity budget

Do not introduce multiple message brokers, vector databases, agent frameworks, browser automation stacks or always-on autonomous agents without measured justification. Prefer one database, one durable queue/workflow strategy, one primary service language and one local inference runtime in the first implementation.

Python MAY be used for specialist ML/data tasks where its ecosystem is materially better. It is not the default language for every service.

## 8. NixOS release and build policy

- Use the latest **released stable** NixOS branch at implementation time, not a future scheduled release.
- At the specification baseline date, NixOS 26.11 was scheduled for late November 2026. The implementation agent MUST verify that it has actually been released before selecting it.
- If 26.11 is not yet stable/released at build time, use the latest released stable branch and record the reason. Do not point release builds at an unreleased branch.
- Pin all flake inputs in `flake.lock`; review every lock update.
- Separate `iso`, `host`, `hardware`, `services` and optional feature modules.
- Use binary caches by default; selectively build from source only when needed.
- Prefer native x86_64 Linux as the supported ISO build host. Windows/WSL2 may be used for editing, but is not the primary release builder.
- ISO build scripts MUST NOT write to block devices.
- Build manifest records Git commit, dirty state, Nix version, nixpkgs revision, architecture, configuration, ISO filename/size/hash and test evidence.

## 9. Repository layout (target)

```text
razstudio-os/
├── flake.nix
├── flake.lock
├── README.md
├── LICENSE
├── .gitignore
├── .editorconfig
├── hosts/
│   └── razstudio-main/
│       ├── default.nix
│       └── hardware-configuration.nix
├── iso/
│   └── default.nix
├── modules/
│   ├── core/
│   ├── hardware/
│   ├── networking/
│   ├── security/
│   ├── storage/
│   ├── recovery/
│   ├── diagnostics/
│   └── services/
│       ├── api/
│       ├── agents/
│       ├── inference/
│       ├── research/
│       ├── knowledge/
│       ├── skills/
│       └── workspaces/
├── packages/
│   ├── scripts/
│   └── service-apps/
├── services/
│   ├── gateway/
│   ├── agent-runtime/
│   ├── model-router/
│   ├── paeSE/
│   ├── pace/
│   ├── drke/
│   ├── skill-governance/
│   └── workspace-runner/
├── models/
│   ├── catalogue/
│   ├── manifests/
│   └── qualification/
├── skills/
│   ├── drafts/
│   ├── approved/
│   ├── retired/
│   └── test-fixtures/
├── docs/
│   ├── HARDWARE.md
│   ├── BUILD.md
│   ├── INSTALL.md
│   ├── RECOVERY.md
│   ├── SECURITY.md
│   ├── OPERATIONS.md
│   └── DECISIONS.md
├── tests/
│   ├── evaluation/
│   ├── vm/
│   ├── integration/
│   └── hardware/
└── artifacts/
```

The spelling of implementation directories may be adjusted by the implementation team, but service boundaries and responsibilities MUST remain clear. Do not create empty directories merely for appearance; create them as their phase begins.

---

# PART V — MODEL STRATEGY AND AGENT DESIGN

## 10. Heterogeneous model strategy — locked

Different agents MUST be independently assignable to different models. There is no global single-model requirement. Model assignments are configuration, not hard-coded agent identity.

Each agent profile declares:

- role and objective;
- selected model/provider policy;
- fallback model/provider;
- context and token budget;
- allowed tools and permissions;
- privacy and network-egress policy;
- latency and cost budget;
- resource class (CPU, GPU, hosted);
- required skills;
- validation requirements;
- retry, timeout and escalation policy.

The model router selects the best available model for each task. It may select a different model for the same agent on a later task. It records model ID, provider, quantization/runtime, prompt/template version, route reason and fallback events for each job.

### 10.1 Initial model profiles — locked baseline, hardware qualification required

The following are **candidate profiles**, not claims that every model has already passed on this machine. The model catalogue MUST store exact repository/checkpoint revision, license, quantization, file size, context configuration, runtime compatibility and checksum.

| Role | Initial candidate | Deployment policy |
|---|---|---|
| General compact agent / tool-oriented assistant | NVIDIA Nemotron 3.5 Bolt E4B 4GB GGUF checkpoint | First experimental candidate; validate exact GGUF, runtime and memory on GTX 1650/P1000 before enabling |
| Coding worker | Qwen2.5-Coder-3B-Instruct, quantized GGUF | Local coding baseline; benchmark against newer candidates before promotion |
| Fast triage / lightweight assistant | Qwen2.5-3B-Instruct, quantized GGUF | Local summarisation, classification and compact structured outputs |
| Embeddings | BAAI/bge-small-en-v1.5 | CPU-first retrieval model; verify model license and multilingual suitability |
| Supervisor | Strong hosted reasoning model selected by router | Hosted by default; provider-neutral |
| Independent reviewer | Strong hosted coding/reasoning model, preferably different family/provider from worker | Invoke for significant changes, high-risk work and disputes |
| Research synthesis | Research-specialist hosted or local model chosen per task | Crawling and evidence acquisition are separate from synthesis |

**Important Bolt E4B caveat:** NVIDIA describes a 4GB-targeted Bolt E4B checkpoint and publishes a 4GB model variant. This does not guarantee that the GTX 1650 or Quadro P1000 is a verified deployment target. NVIDIA's published NIM support matrix lists specific newer GPUs for its supported BF16 profile; the user's older GPUs are not thereby certified. Test the exact GGUF/runtime path and keep Bolt optional until it passes. The model has 8B total parameters and 4B effective parameters; effective parameters are not the same as total weight storage.

**Qwen license caveat:** check the actual license file for the exact selected Qwen2.5 checkpoint and intended use. Do not assume all Qwen checkpoints share the same license.

**Model age caveat:** Qwen2.5 candidates are deliberately a stable, compact baseline from the prior model decision, not a claim that they are the newest or strongest models available. The model catalogue should permit future replacements without architectural changes.

### 10.2 Model deployment policy

- ISO includes model catalogue/manifests and download/verification tooling only; it does not bundle weights or start model servers.
- Model weights are downloaded after installation to `/srv/models`.
- Keep a CPU-only inference path.
- Start with one GPU-heavy generation at a time. Worker concurrency is independent of generation concurrency.
- Use per-device inference endpoints or explicit device selection only after verifying runtime support.
- Do not assume two GPUs can be pooled into one model memory space.
- Do not load two full models concurrently into the same 4 GB GPU without measured admission control.
- Set context limits from actual KV-cache and memory measurements; do not blindly use advertised maximum context.
- Use quantized GGUF where supported and record quantization quality trade-offs.
- Model download requires checksum and license capture.
- Model promotion requires representative task evaluations, not a single benchmark.

### 10.3 Model routing policy

Routing score considers:
1. capability fit for task type;
2. expected quality/reliability on relevant evaluation set;
3. context window and actual memory fit;
4. latency and queue state;
5. cost;
6. privacy classification and allowed egress;
7. provider/model availability;
8. fallback/recovery options.

The router MUST be able to escalate a task to a stronger model when confidence is low, validation fails, the worker is blocked or task risk is high. The escalation reason is recorded. A stronger model is not allowed to override security policy or human approval.

## 11. Agent roles

### 11.1 Chief Reasoning and Supervisor

Responsibilities:
- translate owner requests into objectives and acceptance criteria;
- decompose tasks and identify dependencies;
- assign workers and models;
- allocate file ownership and workspaces;
- monitor durable events and failures;
- resolve conflicts and request evidence;
- decide when to escalate to a stronger model or reviewer;
- integrate completed work only after validation;
- preserve task state and next actions.

The supervisor MUST NOT rely on hidden chain-of-thought. It stores concise explicit plans, decisions, evidence references and rationale. It is event-driven rather than consuming tokens to watch every worker token.

### 11.2 Coding workers A/B/C

Start with three independent worker slots. They may be assigned different models and different coding specialties. They receive:
- one bounded objective;
- acceptance criteria;
- repository revision and scoped worktree;
- file ownership boundaries;
- allowed tools and network permissions;
- resource/time budget;
- required tests and evidence format.

Workers do not share mutable worktrees. Conflicting file ownership is prevented by task decomposition or serialized integration.

### 11.3 Testing and integration worker

May be a dedicated worker slot or a role assigned to one of the three workers. Executes real builds, unit/integration tests, static analysis and regression checks. It reports raw commands, exit codes, environment and artifact references. It must not mark a test passed based on model reasoning alone.

### 11.4 Research worker

Performs search, source acquisition, extraction, repository inspection, deduplication, source ranking, freshness checks and evidence packaging. It runs at low priority by default. It does not automatically modify trusted knowledge or approved skills.

### 11.5 Independent reviewer

Activated for significant changes, security-sensitive code, architectural decisions, disputed research or failed tests. Prefer a different model family/provider from the originating worker where feasible. It independently inspects diffs, evidence and tests. It may reject, request changes or recommend acceptance; it cannot merge protected changes without the required approval policy.

## 12. PAESE — Parallel Agent Execution & Supervision Engine

PAESE coordinates concurrent work.

### 12.1 Concurrency model

- Three independent coding worker processes are the initial target.
- One supervisor process coordinates and reacts to durable events.
- One research worker runs separately at low priority.
- Reviewer role is invoked when needed rather than permanently consuming resources.
- Each worker has its own task context, service identity, workspace, resource limits and logs.
- Multiple workers may progress concurrently even when model generations queue behind one inference endpoint.
- A single model server may support multiple slots, but slots share model weights, VRAM and compute. Do not assume independent GPU throughput.

### 12.2 Task decomposition and ownership

Every multi-agent task MUST define:
- subtask IDs and dependencies;
- acceptance criteria per subtask;
- repository commit/base revision;
- owned files/directories or interface contract;
- shared files requiring serialization;
- integration order;
- test responsibility;
- timeout and cancellation conditions.

Do not assign two workers to edit the same files concurrently unless a deliberate conflict-resolution plan exists.

### 12.3 Event-driven supervision

Events include:
- worker started/heartbeat;
- evidence acquired;
- proposal submitted;
- tool permission denied;
- build/test failed;
- worker blocked;
- file conflict detected;
- task completed;
- resource threshold exceeded;
- worker crashed or timed out.

Supervisor actions: continue, request evidence, reassign, ask a worker to repair, escalate model, invoke reviewer, stop unsafe work, or request owner approval.

### 12.4 Resource scheduling

Priority:
1. interactive API/admin requests;
2. active engineering task and validation;
3. reviewer/integration work;
4. background research, crawling and indexing.

Use bounded queues, per-agent CPU/RAM limits, GPU leases/admission control, token/context budgets, network concurrency limits, timeouts, cancellation, retries with backoff and watchdogs. Preserve SSH and core host responsiveness. Do not allow crawler or model workloads to starve the OS.

### 12.5 Canonical task state machine

Task lifecycle state SHALL be:

```text
created
queued
assigned
running
blocked
waiting_for_provider
waiting_for_approval
recovering
completed
failed
cancelled
```

`CheckpointCreated` is an event and MUST NOT by itself be considered a task-state transition. Terminal states are immutable. An authorized retry SHALL create a new attempt, or an explicitly audited retry transition that preserves the prior attempt and outcome.

Every state transition MUST validate the current state, actor permissions, expected aggregate version and transition preconditions. It MUST record the responsible actor, timestamp and audit event. The task-state update and corresponding event insertion SHALL occur in one PostgreSQL transaction.

Recovery SHALL follow:

```text
running -> recovering
  validate last checkpoint
  validate repository snapshot
  inspect outstanding operations
  revoke stale worker ownership
  restore task context
  emit CheckpointRestored
  queued / assigned -> running
```

Recovery SHALL not claim success unless the checkpoint, repository identity and outstanding-operation state are sufficiently established. It SHALL refuse or block unsafe transitions rather than guessing.

## 13. PACE — Persistent Agent Context Engine

PACE is mandatory. It makes work resumable independently of any model or conversation.

Task records SHALL distinguish task state from execution attempts. A task may emit checkpoints repeatedly while remaining in `running`. Each attempt SHALL have a unique attempt ID, worker identity, model/provider route, repository snapshot and start/end outcome.

Persist at minimum:
- task ID, owner/client identity and objective;
- constraints, scope and acceptance criteria;
- project/repository and base commit;
- current status and timestamps;
- plan and dependency graph;
- completed work and outstanding actions;
- findings with evidence references and confidence;
- decisions and rationale;
- assumptions and unresolved questions;
- blockers and escalation history;
- worker/model/provider assignment;
- artifacts, diffs, logs, commands and test results;
- checkpoints and recovery instructions.

Use PostgreSQL for task metadata, state transitions, event records and audit. Store original files, logs and large artifacts on persistent storage; index extracted content separately. Use atomic writes and versioned artifacts.

Context reconstruction MUST be based on durable task records and relevant evidence, not replay of the entire chat transcript. Summaries are navigation aids and never replace source evidence. A worker restart must resume from the latest committed checkpoint or safely restart its bounded task.

### 13.1 Idempotency and operation ledger

External side effects SHALL use durable operation identifiers and a reconciled operation ledger. An operation retry SHALL reuse the original operation ID rather than allocate a different ID.

The operation ledger SHALL support at least:

```text
prepared -> submitted -> confirmed
                      -> rejected
                      -> unknown -> reconcile
```

The ledger is mandatory for Git merges and protected-branch operations, deployments, infrastructure changes, package publication, skill releases and promotions, notifications and external messages. Hosted-model requests SHALL use provider idempotency where supported.

When an external provider has no idempotency guarantee, an `unknown` or uncertain result SHALL be reconciled before retry. The system SHALL not promise exactly-once execution for arbitrary external systems.

### 13.2 Canonical event model

All modules SHALL use one canonical event schema:

| Field | Purpose |
|---|---|
| `event_id` | Globally unique event identifier |
| `task_id` | Related task |
| `event_type` | Canonical event name |
| `actor` | User, supervisor, worker or system |
| `source_module` | Originating module |
| `occurred_at` | Time the event actually occurred |
| `recorded_at` | Time the event was persisted |
| `aggregate_version` | Task event sequence/version |
| `schema_version` | Event payload schema version |
| `attempt_id` | Associated execution attempt |
| `correlation_id` | Related workflow identifier |
| `causation_id` | Event or request that caused this event |
| `payload` | Validated event-specific data |

The initial canonical catalogue includes:

`TaskCreated`, `TaskQueued`, `TaskAssigned`, `TaskStarted`, `TaskBlocked`, `TaskWaitingForProvider`, `TaskWaitingForApproval`, `CheckpointCreated`, `WorkerStarted`, `WorkerHeartbeat`, `WorkerFailed`, `RecoveryStarted`, `CheckpointRestored`, `RecoveryCompleted`, `OperationSubmitted`, `OperationConfirmed`, `OperationUnknown`, `ApprovalRequested`, `ApprovalGranted`, `ApprovalRejected`, `TaskCompleted`, `TaskFailed`, `TaskCancelled`, and `RepositoryRebased`.

The event log SHALL be append-only. The current task-state projection MAY be maintained for efficient queries. PostgreSQL remains authoritative; LISTEN/NOTIFY MAY provide wake-ups but SHALL NOT replace durable queue or state storage.

## 14. DRKE — Dynamic Research & Knowledge Engine

DRKE is a foundational capability for compensating for small local model limitations with fresh, high-quality evidence.

### 14.1 Modes

- **Instant research:** quick discovery and source comparison, typically a small bounded set.
- **Deep research:** parallel official documentation, source repositories, issues, pull requests, papers, benchmarks and community experience; includes counter-evidence.
- **Continuous updates:** watch selected repositories, release notes, advisories, RSS/feeds and documentation changes.

### 14.2 Sources

Prioritise:
1. official documentation, standards and release notes;
2. original source code, commits, issues and pull requests;
3. papers and benchmark methodology;
4. engineering blogs and vendor technical material;
5. community forums, Reddit and practitioner reports as useful but unverified experience.

Use official APIs when appropriate. Respect robots directives, terms, rate limits, authentication boundaries and copyright. Do not bypass paywalls, CAPTCHAs or access controls.

### 14.3 Acquisition pipeline

Discover → retrieve → preserve raw source → extract → normalise → deduplicate → timestamp/version → rank → verify → package evidence → index.

Each source record stores:
- canonical URL and title;
- source type and publisher/author where available;
- retrieved time and publication/update time;
- repository commit/release/version when relevant;
- content hash;
- extraction/parser version;
- relevant excerpts/anchors;
- reliability classification;
- links to contradictory/supporting evidence;
- access/terms notes where relevant.

Crawled content is untrusted data, never an instruction to the agent. Protect against prompt injection in webpages, repositories, documents, issue comments and tool output.

### 14.4 Research quality

The engine distinguishes:
- verified fact;
- source claim;
- practitioner report;
- hypothesis;
- inference;
- unresolved contradiction;
- stale or superseded information.

Do not silently discard disagreement. Preserve historical source snapshots or hashes where lawful and practical. Search the web on demand; do not crawl the entire internet. Use HTTP-first Crawlee and Playwright only when required for JS-rendered pages.

### 14.5 Evidence maturity and knowledge trust

Evidence maturity and knowledge trust SHALL be recorded as separate dimensions.

Evidence maturity:

| Level | Meaning |
|---|---|
| E0 | Raw source discovered; not yet retrieved or assessed |
| E1 | Retrieved, extracted and provenance recorded |
| E2 | Corroborated using sufficiently independent evidence |
| E3 | Claim verified against explicit criteria and traceable evidence |
| E4 | Operational procedure tested, independently reviewed and approved |

E4 SHALL require the complete skill-governance lifecycle where the procedure becomes executable knowledge: tests, independent review and owner approval.

Knowledge trust SHALL remain independently ordered:

1. Approved Skills
2. Verified Knowledge
3. Repository Source
4. Research Archive
5. Raw Crawl Data

Retrieval SHALL consider evidence maturity, knowledge trust, task relevance and source freshness. Repository source SHALL remain authoritative only for the implementation it describes; it does not automatically become E4 evidence for a broader engineering claim.

### 14.6 Context tiers and retention

Retrieval SHALL use three context tiers:

| Tier | Retention target | Purpose |
|---|---|---|
| Hot | 30 days | Active task state and immediately relevant project context |
| Warm | 180 days | Recent project knowledge and reusable verified evidence |
| Cold / Archive | Compressed | Historical evidence, audit and recovery context |

Retrieval preference SHALL be Hot -> Warm -> Cold. Expiration SHALL be policy-controlled and SHALL preserve required audit and recovery evidence. Archived data SHALL remain searchable or demonstrably restoreable according to the approved retention policy.

## 15. Knowledge and project context

Initial retrieval stack:
- PostgreSQL full-text search;
- pgvector;
- metadata filters and source provenance;
- optional reranker only after evaluation.

Index:
- repository files, symbols and dependencies;
- architecture and API documentation;
- compiler/toolchain/build settings;
- configuration and parameter definitions;
- Git history and decisions;
- test/build outputs;
- external documentation and verified research.

Keep separate trust classes:
1. Research Archive — collected, unverified or disputed.
2. Verified Knowledge — reviewed findings with evidence and freshness.
3. Approved Skills — tested, reviewed, human-approved executable procedures.

Retrieval must return source references and confidence metadata. The context packer selects relevant official docs, code, issues, papers, test logs and counterexamples into a compact context package. It must not fabricate citations or hide missing evidence.

## 16. Skill Development & Governance Engine

Agents MUST be able to request and develop new reusable skills on demand. Learning a skill means acquiring a tested procedure, tools, references and constraints; it does not mean retraining model weights by default.

### 16.1 Skill lifecycle

1. Request: agent or owner identifies a capability gap.
2. Discovery: research agent collects official documentation, examples, known limitations and counterexamples.
3. Draft: create a versioned skill package.
4. Static checks: validate schema, tool permissions, dependencies, scripts and secret handling.
5. Sandbox: execute in an isolated environment against positive, negative and adversarial fixtures.
6. Independent review: another model/agent inspects evidence, safety and correctness.
7. Human approval: owner approves promotion to the trusted registry.
8. Release: publish immutable version with changelog and compatibility metadata.
9. Monitor: record use, failures and regressions.
10. Rollback/retire: restore prior approved version or revoke a compromised skill.

### 16.2 Skill package metadata

Each skill MUST declare:
- unique ID, name and semantic version;
- purpose and non-goals;
- author/source provenance;
- supported agent roles and model assumptions;
- dependencies and tool permissions;
- input/output schema;
- instructions and scripts;
- test fixtures and expected outcomes;
- security classification;
- limitations and failure modes;
- review record and approval identity;
- compatibility and deprecation information.

### 16.3 Trust rules

- Draft skills are not approved skills.
- A webpage or repository cannot grant itself tool permissions.
- Skills cannot expand their own privileges.
- Network, filesystem, shell and secret access are explicitly scoped.
- High-impact skills require owner approval.
- No skill is promoted solely because a model says it works.
- Keep immutable versions and rollback history.

---

# PART VI — API, WORKSPACES, SECURITY AND OPERATIONS

## 17. API and remote workspace

### 17.1 Separate interfaces

**Inference API**
- OpenAI-compatible routes where practical.
- Model listing only exposes models permitted by policy.
- Client keys are scoped and rate-limited.
- Provider credentials remain server-side.

**Agent Workspace API**
- create/list/select authorised workspace;
- submit durable job and receive job ID;
- inspect status, events, progress and blockers;
- retrieve approved logs, diffs and artifacts;
- approve/reject gated actions;
- cancel or pause jobs;
- inspect agent/model routing metadata allowed to that client.

Long-running jobs are asynchronous. Client disconnect does not delete work or cancel it unless explicitly requested.

### 17.2 Client identity and API keys

- Every authorised client receives a unique high-entropy revocable key.
- Raw key is displayed once; store only a verifier/hash.
- Support create, rotate, expire, revoke and per-key quotas.
- Revoking one client does not affect other clients.
- Keys never appear in logs, model prompts, Git or artifacts.
- Client identity is enforced server-side; model instructions cannot override it.

### 17.3 Security boundaries

- LAN-first; no public listener by default.
- Remote access through VPN or explicitly reviewed TLS ingress.
- Firewall default deny for unsolicited inbound traffic.
- SSH key authentication; root SSH login disabled.
- Least-privilege service accounts and filesystem permissions.
- Podman/systemd isolation for tools and workers.
- Read-only repository access by default; write/build/test/network permissions are separate grants.
- Secrets held server-side using suitable secret facilities/systemd credentials.
- Retrieved content and repository instructions are untrusted.
- Audit every privileged action, tool call, approval and external egress.
- Redact secrets from logs and error messages.

### 17.4 Human approval gates

Require explicit owner approval for:
- disk partitioning, formatting, wiping or bootloader changes;
- kernel/driver changes with boot impact;
- firewall changes or public network exposure;
- secret access or rotation;
- privilege escalation;
- production deployments;
- protected branch merges;
- promotion of skills into approved registry;
- destructive repository operations;
- changes that could destroy user data.

No model may self-approve these operations.

## 18. Workspace and code execution isolation

- Each coding job gets its own Git worktree or isolated clone.
- Bind the task to a specific repository and base commit.
- Each task SHALL persist `repository_id`, `repository_identity`, `base_commit_hash`, `base_tree_hash`, `worktree_id`, `worktree_path`, `index_version` and `index_source_revision`.
- A worker SHALL continue against its original snapshot until an explicit rebase is approved.
- A worker SHALL NOT silently change its base commit because another worker moved a branch.
- Repository intelligence SHALL identify which commit its symbol index, dependency graph and test relationships describe.
- Branch names alone SHALL NOT be used to establish repository identity or snapshot consistency.
- A branch name SHALL only be a convenience locator; the actual commit and tree hashes are authoritative.

An approved rebase SHALL:

1. record the previous repository snapshot;
2. record the new base commit and tree hash;
3. reconcile existing changes;
4. invalidate or refresh affected repository indexes;
5. record `RepositoryRebased`;
6. resume with the new approved snapshot.

- Default to read-only source access until write permission is granted.
- Use separate service identities and scoped directories.
- Apply CPU, RAM, disk, process, network and wall-time limits.
- Build/test commands run in controlled environments.
- Preserve stdout/stderr, exit code, environment, commit, diff and artifact hashes.
- Prevent access to host secrets, unrelated repositories and persistent data.
- Cancelled/failed workers must release leases and clean only their own temporary workspace.
- Integration occurs through reviewed patches/commits, not shared mutable working directories.

## 19. Storage and persistence

Candidate persistent paths:

```text
/srv/models
/srv/repositories
/srv/knowledge
/srv/skills
/srv/services
/srv/artifacts
/srv/backups
/srv/workspaces
```

Storage layout:
- EFI System Partition → `/boot` as appropriate;
- OS filesystem → `/`;
- persistent data → separate `/srv` filesystem/subvolume/dataset.

Btrfs is a candidate for snapshots; Ext4 is a valid simpler alternative. LUKS is optional but requires a tested recovery-key procedure. Final selection waits for disk inventory and owner decision.

Important data must not exist only inside rollback-prone root snapshots. Backups must be separate from the source disk. Define retention, encryption, restore drills and integrity checks. Never hardcode `/dev/sda`; never auto-format.

## 20. Observability, backup and recovery

Minimum operational telemetry:
- host CPU/RAM/disk/network/temperature;
- GPU utilisation, memory, temperature and driver health when available;
- model queue depth, TTFT, tokens/sec, errors and active leases;
- per-agent state, retries, failures and resource use;
- crawler concurrency, fetch errors and rate limits;
- database health, queue age and backup age;
- API latency, authentication failures and rate limits;
- skill execution outcomes and version.

Use journald initially; add Prometheus/Grafana after stable metrics contracts. Provide health endpoints and watchdogs. Failed workers must not take down the gateway or database.

Backups:
- PostgreSQL logical/base backups as appropriate;
- persistent knowledge, skills, task state and artifacts;
- configuration and flake lock;
- model manifests and hashes (weights may be re-downloaded if verified);
- encryption and off-host copy where available.

Recovery MUST be tested: restore database, recover task state, restore skills/knowledge, rebuild host and boot previous NixOS generation.

---

# PART VII — INITIAL MODEL / RESOURCE SCHEDULING PROFILE

## 21. Initial resource policy

The machine is a constrained local inference host. It is not a multi-GPU workstation with pooled memory.

Initial policy:
- one active GPU-heavy generation at a time until benchmarked;
- separate worker processes may wait, plan, crawl, inspect code or run CPU tests concurrently;
- CPU inference is fallback and low-priority;
- GTX 1650 is the first GPU candidate for small-model tests;
- Quadro P1000 is a secondary experimental device, not assumed equivalent;
- embedding generation defaults to CPU until measured;
- research crawling and extraction are bounded and low priority;
- interactive request latency is protected;
- model context limits are set from measured KV cache/RAM/VRAM, not model marketing maximums;
- memory pressure triggers admission rejection/backpressure rather than host instability.

The 48 GB host-RAM envelope SHALL be treated as a planning budget, not a reservation. GPU VRAM SHALL be accounted separately. The initial planning ceiling SHALL allocate no more than:

| Component | Planning ceiling |
|---|---:|
| Operating system | 4 GB |
| PostgreSQL | 6 GB |
| Agent Core | 3 GB |
| Knowledge/Retrieval | 6 GB |
| Build/Test | 6 GB |
| Inference | 8 GB |
| Reserved headroom | 15 GB |
| Total | 48 GB |

The 15 GB headroom is uncommitted capacity and SHALL remain available for measured system needs. Service figures are initial ceilings, and actual runtime allocation SHALL be measured and adapted. Controlled bursts may use headroom only while host pressure permits.

Before compromising Agent Core or PostgreSQL, the system SHALL reduce background build, indexing and research activity. APU shared-memory configuration SHALL be measured during hardware qualification. Interactive/admin requests and core database health take precedence over background model and retrieval workloads.

### 21.1 Model qualification gate

For each candidate record:
- exact model repo and immutable revision;
- model architecture and total parameters;
- checkpoint file size and quantization;
- license;
- runtime/version and build flags;
- GPU PCI ID and driver version;
- VRAM and system RAM peak;
- context length tested;
- TTFT and tokens/sec;
- output quality on representative NixOS tasks;
- tool-call and JSON reliability;
- concurrent workload impact;
- restart/recovery behaviour.

A candidate is not “supported” until the exact tested configuration passes. Mark it `candidate`, `experimental`, `qualified` or `retired`.

### 21.2 Initial benchmark suite

- code explanation and patch tasks;
- multi-file code navigation;
- test generation and failure diagnosis;
- structured JSON/tool calling;
- source-grounded research synthesis with citations;
- context reconstruction after worker restart;
- prompt-injection resistance with untrusted webpage/repository content;
- memory/latency under simultaneous worker requests;
- long-running queue fairness and cancellation.

---


# PART XI — LOCKED DECISIONS AND OPEN QUESTIONS

## 36.5 Post-review implementation contracts

### 36.5.1 Logical module organisation

Repository Intelligence and the Resource Controller SHALL initially be logical modules with defined interfaces, not independently deployed services unless measured operations justify the additional process and deployment boundary.

The MVP architecture SHALL remain:

```text
RAZStudio OS
Foundation
NixOS · systemd · Podman · PostgreSQL + pgvector

Gateway API

Agent Core
  Supervisor
  Scheduler
  Worker Manager
  State Manager
  Event Engine
  Checkpoint Manager
  Recovery Manager
  Model Routing

Workspace Runner
  Worktrees
  Build and test execution
  Isolation

Resource Controller
  GPU admission
  RAM budgets
  CPU controls
  Network policy

Repository Intelligence
  Symbol index
  Dependency graph
  Ownership
  Build/test relationships

Inference Layer
  llama.cpp
  Local models
  Hosted models
  Provider adapters

Research Engine (optional)
Skill Engine (optional)
```

### 36.5.2 Implementation order

Post-review implementation SHALL establish, in order:

1. PostgreSQL schema and canonical event model;
2. canonical task state machine;
3. idempotency and operation ledger;
4. Agent Core and Recovery Manager;
5. Resource Controller;
6. repository snapshot and indexing contracts;
7. context tiers and evidence maturity.

Each contract SHALL have acceptance tests, failure-injection tests and recorded evidence before promotion.

## 37. Locked architectural decisions

| ID | Decision | Status |
|---|---|---|
| D-001 | NixOS + flakes, stable release pinned | Locked |
| D-002 | First milestone is minimal headless installable/recoverable ISO | Locked |
| D-003 | AI services and model weights are post-install modules, not bundled into ISO | Locked |
| D-004 | Headless-first; no desktop environment | Locked |
| D-005 | Provider-neutral model router | Locked |
| D-006 | Different agents may use different models independently | Locked |
| D-007 | Three isolated coding worker slots, one supervisor, one research worker; reviewer on demand | Locked baseline |
| D-008 | Event-driven supervisor, not token-by-token observer | Locked |
| D-009 | PACE durable task state and checkpoints | Locked |
| D-010 | DRKE on-demand and opt-in continuous research | Locked |
| D-011 | Skill development requires sandbox, independent review and owner approval for promotion | Locked |
| D-012 | PostgreSQL + pgvector + full-text search initially | Locked baseline |
| D-013 | llama.cpp initial local inference runtime, with CPU fallback | Locked baseline |
| D-014 | Crawlee HTTP-first; Playwright only when needed | Locked baseline |
| D-015 | Podman/systemd isolation; no Kubernetes initially | Locked |
| D-016 | LAN-first; no public listener by default | Locked |
| D-017 | Separate inference API and agent workspace API | Locked |
| D-018 | Unique revocable per-client API keys; upstream provider credentials server-side | Locked |
| D-019 | Separate worktrees and file ownership for concurrent coding agents | Locked |
| D-020 | Human approval for destructive, privileged, public exposure, protected merge and skill promotion | Locked |
| D-021 | Two GPUs remain independent 4 GB devices; no assumed pooled VRAM | Locked |
| D-022 | Initial model candidates: Bolt E4B 4GB, Qwen2.5-Coder-3B-Instruct, Qwen2.5-3B-Instruct, BGE-small-en-v1.5; exact configs require qualification | Locked baseline |
| D-023 | Hosted strong reasoning supervisor and independent reviewer are permitted and preferred for demanding work | Locked |
| D-024 | Stability/recoverability over peak theoretical speed | Locked |
| D-025 | Host is orchestration-first and inference-second; hosted reasoning remains first-class execution capacity | Locked |
| D-026 | Agent Core is the strategic product center; models, GPUs and providers are replaceable execution resources | Locked |
| D-027 | Offline operation preserves task state, repositories, checkpoints, approved skills, verified knowledge and available local inference | Locked |
| D-028 | Recovery correctness over execution speed; uncertain state or external side effects SHALL be blocked/reconciled before retry | Locked |
| D-029 | Repository Intelligence and Resource Controller begin as logical modules with interfaces, not independently deployed services | Locked baseline |
| D-030 | Task state, checkpoint events and execution attempts use separate canonical contracts | Locked |
| D-031 | Repository work binds to commit/tree/worktree/index identity and requires an approved rebase event | Locked |

## 38. Open decisions — must be resolved with evidence

1. Exact NixOS stable branch/revision at build time.
2. Build host OS/Nix version and available disk space.
3. Motherboard BIOS version and UEFI settings.
4. Exact GPU PCI IDs, variants, driver binding and supported runtime.
5. Whether M2_2 is occupied and whether PCI_E4 is available.
6. Actual PCIe negotiated width/speed for each GPU.
7. RAM frequency, channel mode and stability with 3 DIMMs.
8. Storage inventory and which separate disk is safe for test installation.
9. Btrfs vs Ext4 and LUKS decision.
10. LAN addressing, DNS and NIC/Wi-Fi support.
11. Whether Temporal adds enough value to justify its operational footprint.
12. Whether a second vector DB is justified by measured retrieval load.
13. Which hosted providers/models are selected and their cost/privacy policy.
14. Which exact model quantizations pass on GTX 1650 and P1000.
15. Backup destination and retention policy.
16. VPN/remote access choice.
17. Whether legacy BIOS boot is required.
18. License review for every model, tool and skill package before use.

Do not block a generic ISO build on decisions that are not required for ISO generation. Do block destructive installation and production service exposure until relevant decisions are resolved.

---


# PART XIII — REFERENCE SOURCES AND MODEL CATALOGUE NOTES

## 40. Initial source references

These references are starting points. Before implementation, agents MUST re-check the current upstream documentation, release state, model revisions, licenses and hardware support matrix.

- NixOS release notes: https://nixos.org/manual/nixos/unstable/release-notes.html
- NixOS 26.11 release schedule: https://github.com/NixOS/nixpkgs/issues/567567
- MSI B450M MORTAR MAX specifications: https://www.msi.com/Motherboard/B450M-MORTAR-MAX/Specification
- NVIDIA Nemotron 3.5 Bolt E4B model card: https://build.nvidia.com/nvidia/nemotron-3.5-bolt-e4b/modelcard
- NVIDIA Nemotron 3.5 Bolt E4B 4GB technical report: https://research.nvidia.com/labs/nemotron/Nemotron-3.5-Bolt-E4B-4GB/
- NVIDIA NIM support matrix: https://docs.nvidia.com/nim/large-language-models/latest/support-matrix.html
- Qwen2.5-Coder-3B model page: https://huggingface.co/Qwen/Qwen2.5-Coder-3B
- Qwen2.5-Coder-3B-Instruct model page: https://huggingface.co/Qwen/Qwen2.5-Coder-3B-Instruct
- BGE small English v1.5 model page: https://huggingface.co/BAAI/bge-small-en-v1.5

Model and hardware claims must be checked against the exact revision and configuration used. NVIDIA's 4GB-oriented Bolt checkpoint is an experimental candidate for this machine, not a guarantee that either older GPU is an officially verified deployment target. The NIM support matrix and a GGUF runtime path are different deployment claims and must not be conflated.

---

## 41. Definition of done — project-level

NixOS is not complete merely because an ISO builds or an agent can answer a prompt.

The initial ISO milestone is complete only when it boots in a VM and on the target PC, passes diagnostics and safety checks, has a reproducible manifest/checksum, and has a demonstrated recovery path.

The core AI engineering platform is complete only when:
- agents can use different models independently;
- three isolated coding workers can progress on separate tasks;
- the supervisor can detect failures and coordinate recovery;
- task state survives model/worker/provider restarts;
- research results preserve provenance and contradictions;
- knowledge trust tiers are enforced;
- skills can be drafted, tested, reviewed, approved and rolled back;
- remote clients are isolated and audited;
- model/GPU resource use is bounded;
- real builds/tests and independent review provide evidence;
- backup and restore are demonstrated;
- long-running stability tests pass without loss of committed task state.

**Final governing principle:** NixOS must remain useful when a model is replaced, a provider is unavailable, a worker crashes, a GPU fails or the server reboots. Models are replaceable components. Durable state, security boundaries, evidence, reproducibility and recoverability are the foundation.


## Cross-document references

- Product scope and acceptance outcomes: [Requirements.md](../requirements/Requirements.md).
- Implementation sequence and completion evidence: [tasks.md](../implementation/tasks.md).
