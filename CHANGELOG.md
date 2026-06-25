# Changelog

## 2026-06-25

- Restructured the contract around Memory as the core shared surface: added `memory/README.md`, moved the app integration guide to `memory/app-integration.md`, and made root `README.md` a read-order and layout index.
- Moved Chat-specific contract material under `apps/chat/`, including the Chat boundary and JSON examples, with short compatibility notes at the old top-level paths.
- Added the practical app integration call sequence, now located at `memory/app-integration.md`: backend configuration, credential transport, delegated actor headers, bootstrap, ingestion, job polling, retrieval, deletion, and a minimal backend helper.
- Documented `X-Api-Key` as the Memory credential transport when deployment infrastructure already uses the `Authorization` header.
- Fixed the Chat ingestion example to include `content.type: "text"`.
- Clarified that production Chat users must be provisioned as canonical Memory principals before becoming active memory-capable users.
- Added membership synchronization requirements for channel/DM creation, explicit member add/remove, user deactivation, open channel auto-join, and role changes.
- Clarified that bootstrap retries are not sufficient for member removal unless Memory explicitly treats the request as a full membership snapshot and deactivates omitted principals.
- Clarified that owner-containment retrieval must still filter app-bound channel/DM Spaces by Memory-owned read membership; broad team or organization ownership is not enough.

## 2026-06-23

- Resolved stale open questions for MemoryView `ask` and owner-containment `ask` now that Memory marks those initial runtime endpoints as implemented.
- Added owner-containment `search` / `context` / `ask` scope for deriving user/team/organization evidence range from `request_origin`.
- Marked initial MemoryView `search` / `context` / `ask` endpoints as implemented in Memory runtime.
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
