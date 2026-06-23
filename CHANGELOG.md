# Changelog

## 2026-06-23

- Added owner-containment `search` / `context` scope for deriving user/team/organization evidence range from `request_origin`.
- Marked initial MemoryView `search` / `context` endpoints as implemented in Memory runtime, while keeping MemoryView `ask` pending on answerer runtime.
- Clarified that pre-acceptance edits update segment content without resetting the segment idle close window.
- Clarified that Chat's default edit/delete lock is Memory acceptance, not a separate pre-acceptance time window.
- Fixed the default Chat conversation segment idle close window at 72 hours while keeping size thresholds app-configurable.
- Added the Chat Ask API contract with `/ask` endpoints, request shape, grounded natural-language response shape, citation/evidence fields, and insufficient-evidence behavior.
- Fixed app integration HTTP contract: delegated actor headers, `mem_app_live_` AppCredential, `POST /v1/app-bindings/bootstrap`, membership permissions, and Chat Source templates.

## 2026-06-21

- Created the initial shared Memory application contract repository.
- Added the initial Chat to Memory boundary contract.
- Added example Chat ingestion and context request payloads.
- Standardized the participating repository submodule path as root-level `memory_app_contract/`.
- Required app service credential plus delegated `on_behalf_of` semantics for Chat production integration.
- Clarified that MemorySpace is the canonical concept and Chat channels/DMs are app-local bindings.
- Added the conversation segment ingestion model, including message-boundary size cuts, idle closure, sent-message locking, and unsent local context overlay.
