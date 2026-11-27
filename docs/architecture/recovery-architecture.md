# Recovery Architecture

**Status:** Locked by D-028.

Recovery validates checkpoints, repository snapshots and outstanding operations before resuming. Stale worker ownership is revoked. Uncertain external side effects are reconciled before retry.
