# Chat App Contract

Chat is the first application integration for Memory. It is an example app contract, not the center of the shared contract.

Read the core Memory contract first:

- [Memory core contract](../../memory/README.md)
- [Memory app integration guide](../../memory/app-integration.md)

Then use these Chat-specific documents and examples:

- [Chat to Memory boundary](boundary.md)
- [Bootstrap example](examples/chat-bootstrap.json)
- [Ingestion example](examples/chat-ingestion.json)
- [Context request example](examples/chat-context-request.json)
- [Ask request and response example](examples/chat-ask.json)

## Chat local model

Chat owns the chat UI, sessions, threads, messages, attachments, outbox processing, batching, and how returned Memory context is shown or used in prompts.

Memory owns canonical principals, memberships, MemorySpace, MemoryView, Source, Policy, RawEvidence, lifecycle, and authorization semantics.

Chat presents MemorySpace through app-local resources:

- `channel`: shared MemorySpace for group conversation
- `dm`: MemorySpace for DM participants
- `personal_agent_dm`: personal MemorySpace for user-agent conversation

Chat stores app-local projections such as `memory_app_binding_id`, `memory_space_id`, and `memory_source_id`. These are bindings to Memory resources, not canonical Memory resources owned by Chat.

## Chat ingestion model

Chat normally ingests conversation segments, not individual messages. The adapter groups eligible messages from one channel or DM, preserves message boundaries, and sends the segment to:

```http
POST /v1/memory-spaces/{space_id}/ingestions
```

The Chat segment Source uses:

```text
source_type: chat_conversation_segment
source_id: src_chat_segments
```

The segment metadata keeps Chat provenance, including channel, thread, message ids, segment id, close reason, event time, transform, and checksum when available.

After Memory accepts a segment, Chat may lock the included messages from normal edit/delete. Messages not yet accepted by Memory remain app-local and can be edited or deleted according to Chat policy.

## Chat read model

For normal user-facing memory answers, Chat uses Memory `ask`. For inspection, advanced prompts, tools, or debug UI, Chat may use `search` or `context`.

Cross-space retrieval must use MemoryView or owner-containment scope with Memory-side read membership checks. Chat must not pass arbitrary space lists or filter unauthorized Memory evidence after the fact.
