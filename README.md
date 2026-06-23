# Memory App Contract

This repository is the shared integration contract for the Memory System and applications that use it.

The first application contract is Chat:

- [Chat to Memory Boundary](chat-memory-boundary.md)
- [Open Questions](open-questions.md)
- [Bootstrap Example](examples/chat-bootstrap.json)
- [Ingestion Example](examples/chat-ingestion.json)
- [Context Request Example](examples/chat-context-request.json)
- [Ask Request and Response Example](examples/chat-ask.json)

## Source of truth

Memory owns:

- canonical user, app, team, and membership semantics
- app service credential and delegated actor authorization semantics
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

Applications do not own canonical Memory identity. App-local users, channels, DMs, projects, or folders are bindings or projections over Memory-owned principals and resources.

## Fixed app integration contract

The current app integration contract uses:

- `Authorization: Bearer <mem_app_live_...>` for app service credential authentication
- `X-Memory-On-Behalf-Of-Type` and `X-Memory-On-Behalf-Of-Id` for delegated actor context
- `POST /v1/app-bindings/bootstrap` for idempotent app-local resource binding
- `read`, `write`, `delete`, `export`, `admin` as the minimum membership permissions
- `chat_conversation_segment` as the Chat conversation segment `source_type`

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
