# System Overview

**Purpose:** Define the deployable boundaries of RAZStudio OS.
**Status:** Proposed architecture, locked foundations where identified in design.md.
**Dependencies:** NixOS, systemd, Podman, PostgreSQL, pgvector.

Agent Core is the strategic center. It owns durable task orchestration and recovery. The Resource Controller, Workspace Runner, Repository Intelligence, Inference Layer, Research Engine and Skill Engine support it.

The ISO is a lightweight recovery/install artifact and MUST NOT require model weights or active AI services.

Security and failure behavior are defined in [security-model.md](security-model.md) and [recovery-architecture.md](recovery-architecture.md).
