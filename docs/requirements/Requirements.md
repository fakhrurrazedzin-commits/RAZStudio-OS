# RAZStudio OS — Requirements

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


# PART I — PRODUCT DEFINITION

## 1. Purpose and vision

NixOS is a reproducible, modular, headless NixOS-based personal AI engineering and research server. It turns one desktop PC into a centrally managed environment that can:

- host small local AI models and connect to stronger hosted models;
- operate multiple specialised AI agents concurrently on independent tasks;
- route different agents to different models and providers;
- supervise, review and verify agent work;
- research the live web, documentation, source repositories, release notes, issues, papers and community discussions on demand or in the background;
- preserve task state, evidence, decisions and checkpoints across model, worker, provider and server restarts;
- create, test, version and govern reusable agent skills;
- provide secure remote API access and a separate remote agent-workspace interface to authorised Windows, Linux and Android clients;
- isolate coding jobs in separate workspaces and Git worktrees;
- build, test, inspect and report on software projects with traceable evidence.

The design priority is **long-running stability, persistent task context, predictable resource usage, recoverability and verified engineering over peak theoretical speed**. The system should stay responsive during long work and should not lose progress when a model, provider, worker or host restarts.

The system is not intended to imitate a single chat session with a larger prompt. It is a durable engineering environment in which specialised agents collaborate through explicit tasks, controlled tools, versioned knowledge and verifiable artifacts.

## 2. Product principles — locked

1. **Provider-neutral:** no agent or subsystem is permanently tied to one model vendor.
2. **Heterogeneous agents:** different roles MUST be able to use different models independently.
3. **Dynamic routing:** the supervisor/router selects a model based on task capability, context, latency, cost, privacy, reliability and availability.
4. **Persistent state:** task objectives, plans, evidence, decisions, checkpoints, artifacts and next actions live outside model context.
5. **Parallel but isolated:** agents may work simultaneously, but not by sharing mutable worktrees or uncontrolled file ownership.
6. **Evidence before confidence:** claims and code changes require traceable sources, tests or explicit uncertainty.
7. **Research is not truth:** crawled content is untrusted until evaluated; preserve provenance and counter-evidence.
8. **Skill learning is governed:** agents may develop skills on demand, but no unreviewed skill silently becomes trusted or broadly executable.
9. **Human authority:** the owner remains final authority for destructive, privileged, externally exposed or production-impacting actions.
10. **Resource-aware:** interactive work outranks active engineering, which outranks background crawling/indexing.
11. **Graceful degradation:** local inference, GPU acceleration, external APIs or optional services may fail without taking down the host or destroying task state.
12. **No premature complexity:** start with PostgreSQL, systemd, a bounded queue and a small number of services; add infrastructure only when measurements justify it.
13. **No false parallelism claims:** worker processes may operate concurrently while model generation is serialised by GPU/inference admission control.
14. **Reproducible and recoverable:** pinned dependencies, known-good generations, backups, checksums and tested restoration paths.
15. **Recovery correctness:** uncertain task state or external side effects MUST be reconciled before retry; performance MUST NOT outrank state consistency or auditability.
16. **Orchestration-first:** durable orchestration, research and repository workflows remain usable when hosted providers or internet access are unavailable.

## 3. Users, clients and operating model

- Primary administrator: machine owner.
- Client devices: Windows, Linux and Android.
- Primary operation: headless, potentially 24/7.
- Initial network boundary: trusted LAN.
- Remote/off-site access: VPN or explicitly secured HTTPS ingress; public exposure is disabled by default.
- Clients use two separate interfaces:
  1. **Inference API:** OpenAI-compatible endpoint where practical, for chat/completions and supported model calls.
  2. **Agent Workspace API/UI:** submit durable jobs, select repositories/workspaces, inspect progress, approve actions, view logs/diffs/artifacts and cancel work.
- Agents and provider credentials are centrally managed. Client applications do not receive upstream provider keys.
- Multiple authorised clients share the central agent pool under per-client identity, quotas, permissions and audit.

---

# PART II — SCOPE AND RELEASE STRATEGY

## 4. Scope boundaries

### 4.1 First ISO — P0

The first deliverable is deliberately a **minimal, bootable, installable and recoverable headless NixOS ISO**, plus a safe installed-host baseline. It MUST provide:

- reproducible Nix flake and pinned lock file;
- UEFI-first x86_64 bootable ISO;
- console shell, editor, Git and Nix tooling;
- network, CPU, RAM, PCI, GPU, disk, filesystem and SMART diagnostics;
- secure installation guidance and manual target-disk confirmation;
- host modules separated from live ISO modules;
- secure SSH configuration for the installed host, initially LAN-restricted;
- firewall defaults and no public listener;
- persistent data boundary separated from OS rollback;
- rollback and ISO recovery documentation;
- build manifest and SHA-256 checksum;
- VM boot/evaluation/build tests and physical validation plan.

The ISO MAY contain **model catalogue/manifests and hardware-detection scripts** as inert documentation/data. It MUST NOT ship large model weights or automatically launch inference.

### 4.2 Post-install platform — P1/P2

The installed host will progressively add:

- PostgreSQL and durable task/job state;
- model runtime and model router;
- local model profiles and optional hosted provider adapters;
- central agent registry and orchestrator;
- fast interactive and deep engineering lanes;
- PAESE multi-agent supervision;
- PACE persistent context;
- DRKE research and knowledge ingestion;
- repository indexing and retrieval;
- skill development/governance;
- sandboxed tools, Git worktrees and build/test execution;
- remote inference API and agent workspace API;
- per-client credentials, audit, quotas and approvals.

These services are not to be installed in the live ISO merely to make the first milestone appear complete. They are staged modules for the installed host.

### 4.3 Explicit exclusions

- RAZStudio photo editor, RAW decoding, image/video processing and graphics pipeline services.
- Desktop environment or graphical installer.
- Kubernetes or multi-node orchestration in the initial architecture.
- Public unauthenticated services.
- Automatic disk partitioning, formatting or wiping.
- Unbounded web crawling, CAPTCHA/paywall/access-control bypass.
- Automatic skill promotion or autonomous privileged operations.
- Training/fine-tuning model weights as the default mechanism for learning new skills.
- Running multiple large local models concurrently without measured memory and throughput evidence.

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


# PART VIII — NON-FUNCTIONAL REQUIREMENTS

## 22. Quality attributes

### Stability and recoverability
- Host services use systemd restart policies, health checks and bounded retries.
- Database and task state survive model/provider/worker restarts.
- No single model process is a single point of failure for task persistence.
- CPU-only administration and recovery remain available.

### Performance
- Do not optimise peak tokens/sec at the expense of responsiveness.
- Apply backpressure and queue limits.
- Benchmark actual end-to-end latency, not only model generation speed.
- Keep background crawling/indexing below interactive and active engineering work.

### Correctness
- Every code change has a diff and validation evidence.
- Tests are executed, not inferred.
- Independent review is used for high-impact work.
- Contradictory evidence is preserved.
- Uncertainty is explicit.

### Maintainability
- Pin dependencies and record versions.
- Keep service boundaries small.
- Avoid duplicated sources of truth.
- Version APIs, agent profiles, model manifests and skills.

### Privacy
- Clearly label hosted-model calls and external data egress.
- Local/private tasks stay local when policy requires.
- Provider keys never leave the server.
- Per-client audit and revocation are supported.

### Security
- Least privilege, sandboxing, explicit approvals, prompt-injection defence and secret redaction are mandatory.
- No unauthenticated public service.
- No automatic destructive disk action.

---


# PART X — ACCEPTANCE AND RELEASE GATES

## 32. ISO acceptance matrix

| ID | Test | Pass condition |
|---|---|---|
| ISO-001 | Flake evaluation | No evaluation errors on pinned inputs |
| ISO-002 | ISO build | Expected artifact exists and is non-empty |
| ISO-003 | Checksum | SHA-256 independently verifies |
| ISO-004 | Manifest | Commit, dirty state, versions, architecture, hash and test results recorded |
| ISO-005 | VM UEFI boot | Live console reached |
| ISO-006 | VM networking | DNS and HTTPS work |
| ISO-007 | Diagnostics | CPU/RAM/PCI/storage/NIC tools available |
| ISO-008 | No forbidden services | No model, crawler, database or agent service starts in ISO |
| ISO-009 | Physical boot | Target hardware boots USB |
| ISO-010 | Disk safety | No disk write occurs without explicit target confirmation |
| ISO-011 | Recovery | ISO can inspect installed test host |

## 33. Installed-host acceptance matrix

| ID | Test | Pass condition |
|---|---|---|
| HOST-001 | Host boot | Stable boot and console |
| HOST-002 | SSH | Key login works; root denied |
| HOST-003 | Firewall | Default deny and only approved services reachable |
| HOST-004 | Rollback | Previous NixOS generation boots |
| HOST-005 | Persistent data | `/srv` survives OS rollback |
| HOST-006 | NVIDIA fallback | Host boots without NVIDIA acceleration |
| HOST-007 | Backup restore | Database and persistent data restored to test target |
| HOST-008 | Resource health | Host remains responsive under bounded load |

## 34. Agent platform acceptance matrix

| ID | Test | Pass condition |
|---|---|---|
| AG-001 | Client key isolation | Revoking client A does not affect client B |
| AG-002 | API auth | Missing/invalid key rejected |
| AG-003 | API separation | Inference API and workspace API have separate scopes |
| AG-004 | Model routing | Different agents use independently selected models |
| AG-005 | Model fallback | Provider/GPU failure handled without losing task |
| AG-006 | Durable task | Job survives client disconnect and worker restart |
| AG-007 | Three workers | Independent tasks progress concurrently |
| AG-008 | Workspace isolation | Workers cannot modify another worker's workspace |
| AG-009 | File ownership | Conflicting writes prevented or serialized |
| AG-010 | Supervisor | Responds to failure/blocker events |
| AG-011 | Reviewer | High-risk change receives independent review |
| AG-012 | Test evidence | Build/test claims contain real commands and outputs |
| AG-013 | Resource control | GPU/CPU/RAM/queue limits enforce backpressure |
| AG-014 | Cancellation | Authorized client can cancel its job |
| AG-015 | Audit | Client-to-agent-to-model-to-tool-to-result trace exists |
| AG-016 | Secret isolation | Provider keys never exposed to client or logs |
| AG-017 | Task state transitions | Illegal, out-of-version or unauthorised transitions are rejected and state plus audit event commit atomically |
| AG-018 | Idempotent external operations | Crash/retry after an unknown external result creates no duplicate operation and reconciles before reuse |
| AG-019 | Canonical event model | Cross-module task events validate the shared schema and retain attempt/correlation/causation identity |
| AG-020 | Repository snapshot recovery | Recovery restores the exact commit/tree/worktree/index snapshot or blocks until an approved rebase |
| AG-021 | Recovery correctness | D-028 recovery scenarios pause or reconcile unsafe state instead of reporting false success |

## 35. Research, knowledge and skill acceptance matrix

| ID | Test | Pass condition |
|---|---|---|
| RK-001 | Source provenance | Every research claim links to source record |
| RK-002 | Freshness | Retrieval/publication times and version are preserved |
| RK-003 | Contradiction handling | Conflicting sources remain visible |
| RK-004 | Prompt injection | Untrusted content cannot change tool permissions |
| RK-005 | Crawl limits | Concurrency, rate and resource limits enforced |
| RK-006 | Knowledge trust tiers | Archive, verified knowledge and approved skills separated |
| SK-001 | Skill draft | Draft stored with version and provenance |
| SK-002 | Sandbox | Skill runs in isolated environment |
| SK-003 | Negative tests | Failure/adversarial fixtures executed |
| SK-004 | Independent review | Reviewer findings recorded |
| SK-005 | Human approval | Promotion requires owner approval |
| SK-006 | Rollback | Prior approved version can be restored |
| SK-007 | Permission boundary | Skill cannot grant itself additional permissions |
| EV-001 | Evidence maturity | Evidence records distinguish E0-E4 maturity from knowledge trust |
| EV-002 | Repository snapshot provenance | Repository-derived findings identify commit/revision, worktree and index source |

## 36. Stability and performance tests

At minimum:
- 24-hour initial service soak;
- 72-hour follow-up soak;
- multi-day soak before declaring long-running stability;
- repeated worker termination and restart;
- PostgreSQL restart and restore;
- hosted provider outage and rate limiting;
- GPU driver/runtime failure;
- crawler network failure and rate-limit response;
- disk-space pressure and queue backpressure;
- cancellation during tool execution;
- task reconstruction from checkpoints;
- worker crash after a checkpoint;
- crash immediately after an uncertain external side effect;
- concurrent worker repository movement without approved rebase;
- illegal and stale task-state transitions;
- operation-ledger reconciliation after process restart;
- backup integrity and restore drill.

Record:
- p50/p95 API latency;
- model time-to-first-token and tokens/sec;
- queue wait time and age;
- RAM/VRAM peaks;
- GPU/CPU temperatures;
- crawl throughput and error rate;
- RAG relevance;
- task recovery time;
- data loss (must be zero for committed durable state).

Do not set arbitrary performance targets before baseline measurement. Define target SLOs from measured hardware capacity and actual usage.

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

- Architecture and model assignments: [design.md](../architecture/design.md).
- Implementation order and task IDs: [tasks.md](../implementation/tasks.md).
- Acceptance IDs in this file are mandatory release gates; tasks reference them rather than redefining them.
