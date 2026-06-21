# Open Questions

## User, app, team, and membership model

Memory should own canonical identity and membership, but the exact API shape is not finalized. The Memory repository should capture the final decision in an ADR before implementation.

## MemoryView API

Cross-space retrieval should happen through MemoryView, but the public MemoryView API shape is not finalized.

## Chat source taxonomy

The initial contract uses `chat_message_batch` as a provenance-oriented `source_type`. Confirm whether Memory should standardize this as a common source type or leave it app-defined metadata.

## Ask API

The Memory docs describe search, context, and ask as separate surfaces. The initial implemented app contract should use search/context until the ask API is implemented and documented.
