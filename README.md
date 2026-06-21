# Memory App Contract

This repository is the shared integration contract for the Memory System and applications that use it.

The first application contract is Chat:

- [Chat to Memory Boundary](chat-memory-boundary.md)
- [Open Questions](open-questions.md)
- [Ingestion Example](examples/chat-ingestion.json)
- [Context Request Example](examples/chat-context-request.json)

## Source of truth

Memory owns:

- canonical user, app, team, and membership semantics
- MemorySpace creation and lifecycle
- MemoryView search boundaries
- Source registration and trust metadata
- RawEvidence ingestion
- Policy and requested_use filtering
- search, context, ask, feedback, and delete propagation contracts

Applications own:

- app-local UI and interaction flow
- app-local message, session, and thread models
- batching and formatting before ingestion
- deciding when to write to Memory
- deciding how to use returned memory context

## Space naming model

MemorySpace is the canonical collaboration and retrieval boundary. Applications may present a MemorySpace under an app-local name:

- Chat presents a MemorySpace as a channel, DM, or personal-agent DM.
- Another application may present the same MemorySpace as a project, folder, case, repository, or workspace.

The app-local object is a binding to a MemorySpace, not a new canonical space concept.

## Operational model

Each participating repository should include this repository as a submodule. Agents working in either repository must pull the submodule before relying on it, and push contract edits after committing them.

Recommended submodule path:

```text
memory_app_contract
```

## Compatibility rule

The contract should remain app-independent. A Chat payload can identify itself as Chat through Source and metadata, but Memory should not change structural processing because the source application is Chat.
