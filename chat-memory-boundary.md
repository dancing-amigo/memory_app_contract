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
- segmenting ongoing chat messages into conversation batches before ingestion
- preserving chat-local identifiers in metadata
- deciding how returned ContextPacks are used in prompts or UI

## MemorySpace and Chat channel binding

MemorySpace is the canonical concept. Chat exposes a MemorySpace as a channel, DM, or personal-agent DM. Other applications may expose the same MemorySpace through a different app-local object name.

Chat's channel record is therefore an app-local binding to a MemorySpace:

- `channel` maps to a shared MemorySpace for group conversation.
- `dm` maps to a MemorySpace for the DM participants.
- `personal_agent_dm` maps to the user's personal MemorySpace for user-agent conversation.

Chat must not create users, teams, memberships, or MemorySpaces that exist only in Chat. User and space creation initiated from Chat should call Memory control-plane APIs first, then store or refresh the Chat projection/binding returned by Memory.

## Authentication and delegated actor contract

Chat must authenticate to Memory as the Chat application, not as each individual Chat user.

The required model is:

```text
Chat backend
  -> Memory API
     credential: Chat app service credential
     delegated subject: on_behalf_of canonical Memory principal
     target: MemorySpace or MemoryView
```

An app service credential proves the caller is Chat. It does not, by itself, authorize Chat to read or write every MemorySpace.

Every Chat request that reads or writes user, DM, channel, or team memory must carry an explicit delegated subject, equivalent to `on_behalf_of`. Memory uses that delegated subject as the effective principal for membership and resource authorization.

Memory authorization must check:

- Chat app credential status and scope
- Chat app registration / installation / binding
- delegated principal existence
- delegated principal membership or permission on the target MemorySpace or MemoryView
- requested action: read, write, delete, export, or administer
- requested_use for memory-use policy on reads

Per-user Memory API keys stored by Chat are not the production integration model. If the bearer token is API-key-shaped, it is still semantically a Chat app service credential.

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

Chat should normally ingest conversation segments, not each individual message. A conversation segment is an app-local batch of consecutive messages from one Chat channel/DM binding to one MemorySpace.

Initial segment closing rules should be deterministic and app-local:

- close on inactivity after a configured idle window, for example 24 or 48 hours without a new message
- close when a configured message count, character count, or estimated token count threshold is reached
- always cut on message boundaries; Chat must not split a single message across two segments
- if adding one message crosses the size threshold, Chat may either close before that message or include that whole message and close after it; the chosen behavior must be consistent and represented in metadata
- if a single message exceeds the threshold, the segment may exceed the threshold rather than splitting the message

LLM-based topic boundary detection is optional future behavior. The initial contract does not require it.

## Required metadata for Chat ingestion

Chat should include these metadata fields when available:

- `app_id`: stable app identifier, initially `chat`
- `app_space_binding_type`: Chat's app-local name for the MemorySpace binding, for example `channel`, `dm`, or `personal_agent_dm`
- `memory_space_id`: the canonical MemorySpace id targeted by the ingestion
- `chat_channel_id`: app-local channel or DM identifier
- `chat_thread_id`: app-local thread identifier
- `chat_message_ids`: source message identifiers included in the input
- `conversation_segment_id`: app-local segment id when ingesting a batch
- `segment_start_message_id`: first included message id when ingesting a batch
- `segment_end_message_id`: last included message id when ingesting a batch
- `segment_message_count`: number of messages included in the segment
- `segment_char_count`: source character count represented by the segment
- `segment_closed_reason`: for example `idle_timeout`, `size_threshold`, `manual`, or `backfill`
- `speaker_roles`: roles represented in the input, for example `user` or `assistant`
- `event_time`: source-side event time in ISO 8601 format
- `transform`: how the text was produced, for example `conversation_segment_transcript`, `conversation_segment_summary`, `single_message`, or `curated_memory_input`
- `source_uri`: optional app-local deep link or stable reference
- `content_checksum`: optional checksum of the original source text or batch

If Chat sends a curated or summarized input instead of the full original transcript, metadata must make that explicit through `transform`.

After Memory accepts a segment ingestion, Chat may lock the included messages from edit/delete in the app-local UI and API. This avoids requiring Memory to support message-level update/redaction as the default Chat integration path. Messages not yet included in an accepted segment remain app-local and may be edited according to Chat policy.

## Identity and space bootstrap

Chat may pass app-local identifiers as metadata, but canonical memory identity belongs to Memory.

Initial integration should use Memory's existing idempotent control-plane calls:

1. create or resolve the canonical Memory owner
2. create or resolve the required MemorySpace
3. register or resolve the Chat Source in that MemorySpace
4. ingest RawEvidence into that MemorySpace

Until Memory has full user, app, team, and membership APIs, Chat must not treat chat-local users or spaces as the canonical cross-app model. A temporary owner-only development path may exist, but production Chat integration requires app service credential, delegated subject, and Memory-side membership authorization.

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

Chat may combine returned Memory context with unsent app-local context from open or queued conversation segments. This local context overlay is not Memory output and must be filtered by Chat visibility/membership rules before use in prompts or UI. Chat should not send all unsent channel content by default; it should include only recent or relevant local context for the active user and task.

## What Chat must not do

- Chat must not read Memory's internal database.
- Chat must not define canonical MemorySpace, MemoryView, Policy, or membership semantics.
- Chat must not store one Memory API key per Chat user as the production authentication model.
- Chat must not rely on app credential alone to access arbitrary MemorySpaces.
- Chat must not rely on Chat-specific atom types.
- Chat must not expect Memory to change structural processing only because `app_id` is `chat`.
- Chat must not promote assistant or external content into user Profile, Policy, or Procedure memory without Memory policy allowing it.

## What Memory must not do

- Memory must not depend on Chat's database schema.
- Memory must not require Chat-specific payload structure beyond generic content and metadata.
- Memory must not treat an app service credential as blanket access to all user spaces.
- Memory must not skip delegated membership checks for Chat reads or writes.
- Memory must not expose cross-space memory without an explicit MemoryView or equivalent authorization boundary.
- Memory must not return memory blocked by policy for the requested use.

## Versioning

Breaking changes to this contract require:

1. updating this repository
2. updating the Memory repository docs or ADRs if Memory semantics change
3. updating the Chat repository integration docs or implementation
4. recording the change in `CHANGELOG.md`
