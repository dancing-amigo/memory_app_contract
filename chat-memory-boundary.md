# Chat to Memory Boundary

## Status

Draft contract. This document describes the intended boundary between the Chat application and the Memory System.

## Core principle

Chat sends memory-worthy input plus provenance metadata. Memory stores the input in the correct MemorySpace, applies policy, derives memory through its normal pipeline, and returns only memory the requesting app is allowed to use.

Memory must not become a Chat-specific backend. Chat must not become the canonical owner of users, spaces, memberships, or cross-app memory policy.

## Responsibility split

Memory is responsible for:

- canonical owner, user, app, team, and membership model
- MemorySpace lifecycle
- MemoryView lifecycle and cross-space search boundaries
- Source registration, source trust, and source write/use permissions
- RawEvidence storage
- derived memory lineage
- policy filtering by requested_use
- search, context, ask, feedback, and delete propagation
- audit and mutation logs

Chat is responsible for:

- chat UI and interaction behavior
- chat-local thread, session, and message storage
- deciding which messages or batches should be sent to Memory
- formatting several chat messages into one memory-worthy input when appropriate
- preserving chat-local identifiers in metadata
- deciding how returned ContextPacks are used in prompts or UI

## Ingestion contract

Chat should ingest through the generic Memory ingestion API:

```text
POST /v1/memory-spaces/{space_id}/ingestions
```

Chat sends:

- `source_id`: the registered Source for the Chat stream
- `source_type`: a provenance type such as `chat_message_batch`
- `content.text`: the memory-worthy input
- `metadata`: app provenance and traceability fields
- `allowed_uses` / `disallowed_uses`: optional use constraints
- `processing`: optional generic processing hints

Memory treats this as normal RawEvidence. `source_type` and `metadata.app_id` do not create a Chat-specific structural pipeline.

## Required metadata for Chat ingestion

Chat should include these metadata fields when available:

- `app_id`: stable app identifier, initially `chat`
- `chat_thread_id`: app-local thread identifier
- `chat_message_ids`: source message identifiers included in the input
- `speaker_roles`: roles represented in the input, for example `user` or `assistant`
- `event_time`: source-side event time in ISO 8601 format
- `transform`: how the text was produced, for example `single_message`, `batched_messages`, or `curated_memory_input`
- `source_uri`: optional app-local deep link or stable reference
- `content_checksum`: optional checksum of the original source text or batch

If Chat sends a curated or summarized input instead of the full original transcript, metadata must make that explicit through `transform`.

## Identity and space bootstrap

Chat may pass app-local identifiers as metadata, but canonical memory identity belongs to Memory.

Initial integration should use Memory's existing idempotent control-plane calls:

1. create or resolve the canonical Memory owner
2. create or resolve the required MemorySpace
3. register or resolve the Chat Source in that MemorySpace
4. ingest RawEvidence into that MemorySpace

Until Memory has full user, app, team, and membership APIs, Chat must not treat chat-local users or spaces as the canonical cross-app model.

## Query contract

Chat should query Memory through generic APIs:

```text
POST /v1/memory-spaces/{space_id}/search
POST /v1/memory-spaces/{space_id}/context
```

When MemoryView APIs are available, Chat should prefer MemoryView for cross-space retrieval:

```text
POST /v1/memory-views/{view_id}/search
POST /v1/memory-views/{view_id}/context
```

Every query must include a `requested_use` that describes how the result will be used. Memory must not return memory that policy does not allow for that use.

## What Chat must not do

- Chat must not read Memory's internal database.
- Chat must not define canonical MemorySpace, MemoryView, Policy, or membership semantics.
- Chat must not rely on Chat-specific atom types.
- Chat must not expect Memory to change structural processing only because `app_id` is `chat`.
- Chat must not promote assistant or external content into user Profile, Policy, or Procedure memory without Memory policy allowing it.

## What Memory must not do

- Memory must not depend on Chat's database schema.
- Memory must not require Chat-specific payload structure beyond generic content and metadata.
- Memory must not expose cross-space memory without an explicit MemoryView or equivalent authorization boundary.
- Memory must not return memory blocked by policy for the requested use.

## Versioning

Breaking changes to this contract require:

1. updating this repository
2. updating the Memory repository docs or ADRs if Memory semantics change
3. updating the Chat repository integration docs or implementation
4. recording the change in `CHANGELOG.md`
