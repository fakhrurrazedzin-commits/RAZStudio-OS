# Resource Controller

**Status:** Logical module for MVP; independent deployment is not assumed.

The Resource Controller manages GPU leases, VRAM admission, RAM budgets, CPU limits, network policy and queue control. Workers MUST request resources rather than acquire them independently.
