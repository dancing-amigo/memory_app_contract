# Chat to Memory Boundary

## Status

Stabilized v1 contract. This document describes the intended boundary between the Chat application and the Memory System.

## Core principle

Chat sends memory-worthy input plus provenance metadata. Memory stores the input in the correct MemorySpace, applies policy, derives memory through its normal pipeline, and returns only memory the requesting app is allowed to use.

Memory must not become a Chat-specific backend. Chat must not become the canonical owner of users, spaces, memberships, or cross-app memory policy.

This contract is stable enough for Chat implementation. Future changes must be backward compatible unless this repository records a breaking contract change.

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
- using Memory Ask responses for normal user-facing memory answers
- deciding how returned Search results or ContextPacks are used in advanced prompts, tools, debug views, or UI

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

Delegation is transported in typed HTTP headers:

```http
Authorization: Bearer <app_service_credential>
X-Memory-On-Behalf-Of-Type: user
X-Memory-On-Behalf-Of-Id: user_001
X-Memory-App-Binding-Id: bind_chat_channel_001
```

`X-Memory-App-Binding-Id` is optional during first bootstrap and recommended after a binding exists.

Memory authorization must check:

- Chat app credential status and scope
- Chat app registration / installation / binding
- delegated principal existence
- delegated principal membership or permission on the target MemorySpace or MemoryView
- requested action: read, write, delete, export, or administer
- requested_use for memory-use policy on reads

Per-user Memory API keys stored by Chat are not the production integration model. The production app service credential is an API-key-shaped bearer token with `mem_app_live_` prefix. If Memory reuses API key hashing internally, the token is still semantically a Chat app service credential, not a user key.

Initial app credential scopes are:

- `memory:bootstrap`
- `memory:read`
- `memory:write`
- `memory:delete`
- `memory:export`
- `memory:admin`

Scopes authorize API families. They do not replace MemorySpace or MemoryView membership checks.

## Ingestion contract

Chat should ingest through the generic Memory ingestion API:

```text
POST /v1/memory-spaces/{space_id}/ingestions
```

Chat sends:

- `source_id`: the registered Source for the Chat stream
- `source_type`: `chat_conversation_segment`
- `content.text`: the memory-worthy input
- `metadata`: app provenance and traceability fields
- `allowed_uses` / `disallowed_uses`: optional use constraints
- `processing`: optional generic processing hints

Memory treats this as normal RawEvidence. `source_type` and `metadata.app_id` do not create a Chat-specific structural pipeline.

Chat should normally ingest conversation segments, not each individual message. A conversation segment is an app-local batch of consecutive messages from one Chat channel/DM binding to one MemorySpace.

Initial segment closing rules should be deterministic and app-local:

- close on inactivity after a configured idle window; the default Chat idle window is 72 hours without a new message in the channel/DM
- close when a configured message count, character count, or estimated token count threshold is reached
- always cut on message boundaries; Chat must not split a single message across two segments
- pre-acceptance edits update the transcript content that Chat will ingest, but do not reset the segment idle window
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

After Memory accepts a segment ingestion, Chat may lock the included messages from edit/delete in the app-local UI and API. This avoids requiring Memory to support message-level update/redaction as the default Chat integration path. Messages not yet included in an accepted segment remain app-local; the default Chat policy allows normal edit/delete until Memory accepts the containing segment. Any product-specific pre-acceptance edit window is independent of Memory correctness.

## Identity and space bootstrap

Chat may pass app-local identifiers as metadata, but canonical memory identity belongs to Memory.

Production integration should use the app binding bootstrap API:

```http
POST /v1/app-bindings/bootstrap
Authorization: Bearer <app_service_credential>
X-Memory-On-Behalf-Of-Type: user
X-Memory-On-Behalf-Of-Id: user_001
Idempotency-Key: chat:channel:ch_123
```

Request body:

```json
{
  "app_resource": {
    "type": "channel",
    "id": "ch_123",
    "display_name": "Project Planning",
    "source_uri": "chat://channels/ch_123",
    "metadata": {
      "workspace_id": "ws_001"
    }
  },
  "memory_space": {
    "id": "space_chat_channel_ch_123",
    "name": "Project Planning",
    "owner": { "type": "team", "id": "team_001" },
    "memory_pack_id": "company_agent_v1",
    "storage_binding_id": "storage_team_001",
    "policy_profile_id": "team_private_memory_v1",
    "model_profile_id": "default_ja_memory_v1"
  },
  "memberships": [
    {
      "principal": { "type": "user", "id": "user_001" },
      "permissions": ["read", "write", "delete", "admin"]
    }
  ],
  "source_template": "chat_workspace_segment_source_v1"
}
```

The bootstrap response returns the resolved app binding, MemorySpace, Source, memberships, and created/resolved flags.

Temporary owner-only development integration may use Memory's existing idempotent control-plane calls:

1. create or resolve the canonical Memory owner
2. create or resolve the required MemorySpace
3. register or resolve the Chat Source in that MemorySpace
4. ingest RawEvidence into that MemorySpace

Until Memory has full user, app, team, and membership APIs, Chat must not treat chat-local users or spaces as the canonical cross-app model. A temporary owner-only development path may exist, but production Chat integration requires app service credential, delegated subject, and Memory-side membership authorization.

## Membership permissions

Minimum Memory resource permissions are:

- `read`: search, context, view read, job status, non-export read APIs
- `write`: ingestion and feedback
- `delete`: delete propagation and redaction requests
- `export`: export APIs
- `admin`: Source, membership, app binding, policy, MemorySpace, and MemoryView configuration

Permissions are independent. `admin` does not automatically grant content `read`, `write`, `delete`, or `export`.

## Chat Source templates

All Chat conversation segment Sources use:

```json
{
  "id": "src_chat_segments",
  "source_type": "chat_conversation_segment",
  "display_name": "Chat conversation segments"
}
```

For personal-agent DM or direct user-agent memory with `personal_diary_v1`, use `chat_personal_segment_source_v1`:

```json
{
  "trust_zone": 1,
  "source_authority": "user_utterance",
  "allowed_memory_writes": [
    "observation",
    "decision",
    "open_loop",
    "preference_evidence",
    "profile_candidate",
    "policy_candidate",
    "procedure_candidate"
  ],
  "disallowed_memory_writes": [
    "user_profile",
    "user_preference",
    "policy",
    "procedure"
  ],
  "default_allowed_uses": [
    "search",
    "answer_generation",
    "reflection",
    "project_context"
  ],
  "default_disallowed_uses": [
    "training",
    "external_share",
    "auto_action"
  ],
  "default_sensitivity": "normal"
}
```

For channel, group DM, or workspace/team memory with `company_agent_v1`, use `chat_workspace_segment_source_v1`:

```json
{
  "trust_zone": 2,
  "source_authority": "workspace_member_conversation",
  "allowed_memory_writes": [
    "observation",
    "decision",
    "open_loop",
    "preference_evidence",
    "policy_candidate",
    "procedure_candidate"
  ],
  "disallowed_memory_writes": [
    "policy",
    "procedure"
  ],
  "default_allowed_uses": [
    "search",
    "answer_generation",
    "reflection",
    "project_context"
  ],
  "default_disallowed_uses": [
    "training",
    "external_share",
    "auto_action",
    "profile_update",
    "company_policy_lookup"
  ],
  "default_sensitivity": "normal"
}
```

Official policy documents, admin-authored procedures, imported knowledge bases, or external participants must use different Source templates.

## Query contract

Chat should query Memory through generic APIs with distinct levels of abstraction:

```text
POST /v1/memory-spaces/{space_id}/search
POST /v1/memory-spaces/{space_id}/context
POST /v1/memory-spaces/{space_id}/ask
```

For cross-space retrieval, Chat should prefer MemoryView:

```text
POST /v1/memory-views/{view_id}/search
POST /v1/memory-views/{view_id}/context
POST /v1/memory-views/{view_id}/ask
```

Memory's 2026-06-23 initial MemoryView runtime implements `search` and `context`. `ask` remains part of the accepted contract but depends on the Memory answerer runtime.

When Chat wants Memory to derive the cross-space evidence range from the request origin, it can use owner-containment scope:

```text
POST /v1/memory-scopes/owner-containment/search
POST /v1/memory-scopes/owner-containment/context
```

The request body includes `request_origin`, for example `{ "type": "user", "id": "user_001" }` or `{ "type": "team", "id": "team_001" }`. A user origin resolves the user's own MemorySpaces plus containing team / organization MemorySpaces. A team origin resolves team plus containing organization MemorySpaces and does not include member personal MemorySpaces by default.

`ask` is the primary API for normal user-facing Chat questions that should receive a natural-language answer.

`search` returns matching memory objects, primarily `MemoryAtom` hits with score, route, policy decision, and trace metadata. Chat should use `search` when it needs raw memory records for UI cards, inspection, manual selection, or custom application logic.

`context` returns a `ContextPack`: direct evidence, related memories, summaries, claims, counter-evidence, source refs, provenance, warnings, and retrieval trace. Chat should use `context` when it wants to run its own answerer, perform tool/action grounding, show an evidence/debug view, or inspect why `ask` answered the way it did.

`ask` assembles or reuses a ContextPack, generates a grounded answer inside Memory, and returns the answer plus audit fields. Memory must not answer from evidence that was filtered out by resource authorization or memory-use policy.

Every query must include a `requested_use` that describes how the result will be used. Memory must not return memory that policy does not allow for that use.

For normal user-facing natural-language answers, Chat should use `requested_use: "answer_generation"` unless a more specific registered use is required by the MemoryPack or Policy. If the requested use is not registered or is not allowed, Memory must fail closed or return no usable evidence without revealing blocked memory.

Minimum `ask` request body:

```json
{
  "query": "What should the assistant remember before helping schedule the project planning session?",
  "requested_use": "answer_generation",
  "limit": 8,
  "include_context": false
}
```

Minimum successful `ask` response body:

```json
{
  "answer_id": "ans_001",
  "memory_resource": {
    "type": "memory_space",
    "id": "space_chat_channel_ch_project_planning"
  },
  "query": "What should the assistant remember before helping schedule the project planning session?",
  "requested_use": "answer_generation",
  "answer": {
    "status": "answered",
    "text": "The team generally treats weekends as unavailable for planning sessions, and weekday afternoons are preferred.",
    "language": "en"
  },
  "missing_evidence": false,
  "cannot_answer_reason": null,
  "confidence": "high",
  "context_pack_id": "ctx_001",
  "used_evidence_ids": ["mem_001", "mem_002"],
  "citations": [
    {
      "evidence_type": "memory_atom",
      "evidence_id": "mem_001",
      "source_refs": [
        {
          "raw_evidence_id": "raw_001",
          "source_id": "src_chat_segments"
        }
      ]
    }
  ],
  "warnings": [],
  "trace_id": "trace_001"
}
```

If Memory cannot answer from allowed evidence, it should return an `answer.status` of `insufficient_evidence`, `missing_evidence: true`, an explanatory `cannot_answer_reason`, empty `used_evidence_ids`, and no fabricated answer. Authorization failures should still use the normal API error path rather than an `ask` answer body.

When `include_context` is `true`, Memory may include the assembled `context_pack` alongside `context_pack_id`. The default response should keep the full ContextPack out of the normal Chat payload and expose it through the separate `context` API or trace/debug workflows.

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
