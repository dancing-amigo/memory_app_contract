# Open Questions

## User, app, team, and membership model

Memory should own canonical identity and membership, but the exact API shape is not finalized. The Memory repository should capture the final decision in an ADR before implementation.

The contract assumes Chat will not maintain users, teams, memberships, or MemorySpaces that exist only in Chat. The remaining open question is the exact idempotent control-plane API shape Chat should call when its UI initiates user, space, or membership changes.

## MemoryView API

Cross-space retrieval should happen through MemoryView, but the public MemoryView API shape is not finalized.

## Chat source taxonomy

The initial contract uses `chat_message_batch` as a provenance-oriented `source_type`. Confirm whether Memory should standardize this as a common source type or leave it app-defined metadata.

## Segment thresholds

Chat should ingest conversation segments and cut them only on message boundaries. The exact default idle window and size threshold are not finalized. Candidate defaults are 24 or 48 hours for idle close, plus a character/token/message-count threshold for long uninterrupted conversations.

## Locked message exceptions

The default Chat policy can make messages immutable after their segment is accepted by Memory. Any exceptional delete/redaction workflow for legal, security, or administrative requirements still needs a separate policy decision.

## Ask API

The Memory docs describe search, context, and ask as separate surfaces. The initial implemented app contract should use search/context until the ask API is implemented and documented.
