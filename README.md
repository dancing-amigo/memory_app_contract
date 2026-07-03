# Memory App Contract

This repository is the shared integration contract between the Memory System and applications that use it.

## Read Order

1. [Memory API contract](memory/README.md)
2. [Chat app contract](apps/chat/README.md), when working on Chat integration
3. [Changelog](CHANGELOG.md), when checking compatibility changes

## Repository Layout

```text
memory/             App-facing Memory API contract and generic examples.
apps/chat/          Chat-specific mapping onto Memory primitives.
CHANGELOG.md        Contract change history.
AGENTS.md           Agent workflow rules for this contract repo.
```

## Source Of Truth

Memory owns canonical principals, memberships, MemorySpaces, MemoryViews, Sources, RawEvidence ingestion, policy filtering, and the read/write API contracts.

Applications own their UI, local messages/files/projects/channels, batching before ingestion, and how returned Memory context is used.

App-local resources are bindings to Memory-owned resources. They must not redefine canonical Memory identity, membership, policy, or space semantics.

## Compatibility Rule

Keep the core contract app-independent. App-specific behavior belongs under `apps/<app_id>/`. If an app needs a new Memory API shape, update `memory/README.md` and the generic examples first, then document only the app-specific mapping under the app directory.
