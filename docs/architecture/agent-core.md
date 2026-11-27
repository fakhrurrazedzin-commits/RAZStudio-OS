# Agent Core

**Status:** Locked modular service; implementation pending.

Agent Core SHALL be a modular deployable service containing Supervisor, Scheduler, Worker Manager, State Manager, Event Engine, Checkpoint Manager, Recovery Manager and Model Routing.

It owns durable task orchestration. Models and providers are replaceable resources. Task state and events are authoritative in PostgreSQL.
