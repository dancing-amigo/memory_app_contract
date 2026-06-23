# Open Questions

## User, app, team, and membership model

Memory owns canonical identity and membership. The Memory repository accepted this direction in ADR 0009.

The contract assumes Chat will not maintain users, teams, memberships, or MemorySpaces that exist only in Chat. The idempotent bootstrap API shape is fixed in `chat-memory-boundary.md`.

## MemoryView API

Cross-space retrieval should happen through MemoryView, but the public MemoryView API shape is not finalized.

## Segment size thresholds

Chat should ingest conversation segments and cut them only on message boundaries. The default idle close window is fixed at 72 hours without a new message in the channel/DM. Exact production size thresholds remain tunable per app, with message-count, character-count, or token-count thresholds allowed for long uninterrupted conversations.

## Locked message exceptions

The default Chat policy can make messages immutable after their segment is accepted by Memory. Any exceptional delete/redaction workflow for legal, security, or administrative requirements still needs a separate policy decision.
