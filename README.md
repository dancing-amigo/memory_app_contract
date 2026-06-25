# Memory App Contract

This repository is the shared integration contract for the Memory System and applications that use it.

Memory is the center of this contract. App contracts describe how each application binds its local concepts to Memory-owned primitives without redefining Memory semantics.

## Read order

Agents integrating an application with Memory should read:

1. [Memory core contract](memory/README.md)
2. [Memory app integration guide](memory/app-integration.md)
3. The relevant app contract under [apps/](apps/README.md)
4. [Open questions](open-questions.md)

The first app contract is Chat:

- [Chat app contract](apps/chat/README.md)
- [Chat to Memory boundary](apps/chat/boundary.md)
- [Chat principal and membership contract](apps/chat/principals-and-membership.md)
- [Chat bootstrap example](apps/chat/examples/chat-bootstrap.json)
- [Chat ingestion example](apps/chat/examples/chat-ingestion.json)
- [Chat context request example](apps/chat/examples/chat-context-request.json)
- [Chat ask request and response example](apps/chat/examples/chat-ask.json)

## Repository layout

```text
memory/            Core Memory-owned API and integration contract for every app.
apps/              App-specific contracts. Chat is the first app.
open-questions.md  Unresolved cross-repository integration questions.
CHANGELOG.md       Contract change history.
```

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

## Operational model

Each participating repository should include this repository as a submodule. Agents working in either repository must pull the submodule before relying on it, and push contract edits after committing them.

Recommended submodule path:

```text
memory_app_contract
```

## Compatibility rule

The contract should remain app-independent. A Chat payload can identify itself as Chat through Source and metadata, but Memory should not change structural processing because the source application is Chat.
