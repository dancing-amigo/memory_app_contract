# App Contracts

This directory contains app-specific contracts layered on top of the core Memory contract in `../memory/`.

Each app directory should describe:

- what the app is and which local resources bind to MemorySpace or MemoryView
- which app-local events or records become memory-worthy input
- Source types, required metadata, and examples for ingestion
- app-local batching, locking, retry, and deletion behavior
- how the app uses `search`, `context`, and `ask`
- anything the app must not expect Memory to do structurally for that app alone

App contracts must not redefine Memory internals. If an app needs a new Memory API, data model, policy, evaluation, or membership behavior, update the Memory repository docs or ADRs in the same work and record any unresolved question in `../open-questions.md`.

Current app contracts:

- [Chat](chat/README.md)
- [Google Workspace](google-workspace/README.md)
