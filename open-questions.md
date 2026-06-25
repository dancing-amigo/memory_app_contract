# Open Questions

## User, app, team, and membership model

Memory owns canonical identity and membership. The Memory repository accepted this direction in ADR 0009.

The contract assumes Chat will not maintain users, teams, memberships, or MemorySpaces that exist only in Chat. The idempotent bootstrap API shape is fixed in `apps/chat/boundary.md`.

Production readiness now depends on Memory exposing canonical principal create/resolve behavior that Chat can call before activating a memory-capable local user. A local-only Chat user must be treated as pending or degraded, not as a delegated Memory principal.

Membership removal also needs explicit production semantics. Either app binding bootstrap must support a clearly marked full membership snapshot that deactivates omitted principals, or Memory must expose an explicit membership revoke/full-reconcile API. Additive bootstrap alone is not enough for Chat channel membership correctness.

Owner-containment retrieval must not treat a broad team or organization owner as permission to include every app-bound Chat channel or DM Space. Cross-space retrieval for Chat-visible knowledge must be filtered by Memory-owned read membership on each app-bound collaboration Space.

## Query API status

Resolved for initial runtime: Memory exposes `search`, `context`, and `ask` for single MemorySpace, MemoryView, and owner-containment query scope.

Remaining production work lives in the Memory repository: broader delegated Chat ingestion/query integration tests, answer quality evaluation, and no-leak tests for MemoryView and owner-containment `ask`.

## Segment size thresholds

Chat should ingest conversation segments and cut them only on message boundaries. The default idle close window is fixed at 72 hours without a new message in the channel/DM. Exact production size thresholds remain tunable per app, with message-count, character-count, or token-count thresholds allowed for long uninterrupted conversations.

## Locked message exceptions

The default Chat policy can make messages immutable after their segment is accepted by Memory. Any exceptional delete/redaction workflow for legal, security, or administrative requirements still needs a separate policy decision.
