# Changelog

## 2026-06-23

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
