# RAZStudio OS

RAZStudio OS is a NixOS-based, orchestration-first platform for durable AI-assisted engineering and research. Agent Core owns task state and recovery; models, GPUs and hosted providers are replaceable execution resources.

**Status:** specification and documentation repository. No operating-system implementation or production application code is included.

## Specification

- [Requirements](docs/requirements/Requirements.md)
- [Architecture design](docs/architecture/design.md)
- [Implementation tasks](docs/implementation/tasks.md)
- [Architecture decisions](docs/decisions/README.md)
- [Documentation index](docs/README.md)

## Governing principles

- Orchestration, persistent state and recovery are first-class capabilities.
- Hosted reasoning remains supported and provider-neutral.
- Repository and external side effects are tracked by commit and operation identity.
- Unverified hardware, model and benchmark claims must remain labelled.
- Human approval gates privileged and destructive operations.

See [SECURITY.md](SECURITY.md) and [CONTRIBUTING.md](CONTRIBUTING.md) before proposing implementation changes.
