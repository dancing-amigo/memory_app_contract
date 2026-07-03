# Chat App Contract

This file records only the Chat-specific mapping onto the Memory API contract. Read [Memory API Contract](../../memory/README.md) first.

## Local Resource Mapping

Chat presents MemorySpaces through app-local resources:

- `channel`: shared MemorySpace for group conversation.
- `dm`: MemorySpace for DM participants.
- `personal_agent_dm`: personal MemorySpace for a user-agent conversation.

Chat stores app-local projections such as `memory_app_binding_id`, `memory_space_id`, and `memory_source_id`. These are bindings to Memory resources, not canonical Memory resources owned by Chat.

## Bootstrap

Chat uses the generic Memory bootstrap endpoint:

```http
POST /v1/app-bindings/bootstrap
```

The app-local resource is usually a `channel`, `dm`, or `personal_agent_dm`. Chat should persist the returned app binding id, MemorySpace id, and Source id.

During first bootstrap, `X-Memory-App-Binding-Id` is absent. After bootstrap, app-bound Memory calls should send it.

## Source

Chat conversation segments use:

```text
source_id: src_chat_segments
source_type: chat_conversation_segment
```

Chat bootstrap may use the Chat source templates `chat_personal_segment_source_v1` or `chat_workspace_segment_source_v1` when Memory supports them. Otherwise it can pass an equivalent generic `source` definition through the standard bootstrap request.

Conversation segments are evidence, not canonical policy or procedure documents. Official policy documents, admin-authored procedures, imported knowledge bases, and external participant material should use a different Source.

## Ingestion Unit

Chat normally ingests conversation segments, not individual messages.

Example: [chat-ingestion.json](examples/chat-ingestion.json)

Segment rules:

- Group eligible messages from one channel or DM into a coherent text segment.
- Preserve message boundaries.
- Close on a deterministic idle window, size threshold, manual action, or backfill boundary.
- Default idle close window is 24 hours without a new message.
- Do not split a single message across segments.
- If bounded thread context is needed, include it as context text and mark it separately in metadata.

After Memory accepts a segment, Chat may lock included messages from normal edit/delete. Messages not yet accepted by Memory remain app-local and can be edited or deleted according to Chat policy.

## Required Metadata

Chat should include these fields when available:

- `app_id`
- `app_space_binding_type`
- `memory_space_id`
- `chat_channel_id`
- `chat_thread_id`
- `chat_message_ids`
- `chat_context_message_ids`
- `conversation_segment_id`
- `segment_start_message_id`
- `segment_end_message_id`
- `segment_message_count`
- `segment_char_count`
- `context_message_count`
- `context_char_count`
- `context_truncated`
- `context_depth_limit`
- `context_char_limit`
- `segment_closed_reason`
- `speaker_roles`
- `event_time`
- `transform`
- `source_uri`
- `content_checksum`

If Chat sends curated or summarized input instead of source transcript text, `transform` must make that explicit.

## Membership Sync

Chat channel/DM membership and Memory resource membership are separate records, but they must not diverge in production.

Every Chat membership transition that changes who can read or write a channel or DM must be reflected in Memory:

- channel or DM creation
- explicit member add
- explicit member remove or leave
- user deactivation
- automatic join to an open channel
- role changes that affect Memory permissions

Open channel auto-join is still a membership change. Chat must enqueue durable Memory synchronization for it.

Retrying bootstrap with a shorter membership array is not enough to remove access unless the Memory endpoint explicitly treats the request as a full membership snapshot. Otherwise Chat must call the relevant Memory membership revoke or reconcile API.

## Reads

For normal user-facing memory answers, Chat uses Memory `ask`.

For prompt context, tool/action grounding, inspection, or debug UI, Chat uses Memory `context`.

For raw memory cards, manual selection, or app-specific logic, Chat uses Memory `search`.

Cross-space retrieval must use MemoryView or owner-containment scope with Memory-side read membership checks. Chat must not pass arbitrary space lists or filter unauthorized Memory evidence after the fact.

## Chat Must Not

- Read Memory's internal database.
- Store one Memory API key per Chat user as the production model.
- Treat Chat-local users, channels, DMs, teams, or memberships as canonical Memory resources.
- Rely on app credential alone to access arbitrary MemorySpaces.
- Promote assistant or external content into user Profile, Policy, or Procedure memory unless Memory policy allows it.
- Expect Memory to branch structurally just because `app_id` is `chat`.
