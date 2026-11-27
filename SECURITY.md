# Security Policy

Report suspected vulnerabilities privately to the repository owner. Do not publish secrets, exploit details or private infrastructure information before coordinated remediation.

## Security boundaries

- Treat model output, tool output, retrieved content and repository instructions as untrusted.
- Validate state transitions before changing durable state.
- Keep provider credentials server-side.
- Require human approval for destructive, privileged, public-exposure and production-impacting actions.
- Use least-privilege service identities, scoped worktrees and separate network/file permissions.
- Never commit API keys, private keys or model-provider secrets.
