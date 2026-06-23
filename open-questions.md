# Open Questions

## User, app, team, and membership model

Memory owns canonical identity and membership. The Memory repository accepted this direction in ADR 0009.

The contract assumes Chat will not maintain users, teams, memberships, or MemorySpaces that exist only in Chat. The idempotent bootstrap API shape is fixed in `chat-memory-boundary.md`.

## MemoryView API

Cross-space retrieval should happen through MemoryView, but the public MemoryView API shape is not finalized.

## Segment thresholds

Chat should ingest conversation segments and cut them only on message boundaries. The exact default idle window and size threshold are not finalized. Candidate defaults are 24 or 48 hours for idle close, plus a character/token/message-count threshold for long uninterrupted conversations.

## Locked message exceptions

The default Chat policy can make messages immutable after their segment is accepted by Memory. Any exceptional delete/redaction workflow for legal, security, or administrative requirements still needs a separate policy decision.

## Ask API

The Memory docs describe search, context, and ask as separate surfaces. The initial implemented app contract should use search/context until the ask API is implemented and documented.
