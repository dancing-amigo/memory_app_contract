# Memory Core Contract

This directory records the Memory-owned parts of the app integration contract. Every app integration must treat these concepts, APIs, and authorization boundaries as the shared contract before reading its app-specific notes under `../apps/`.

## Contract authority

Memory owns:

- canonical user, app, team, organization, system, and membership semantics
- app service credential and delegated actor authorization semantics
- MemorySpace creation and lifecycle
- MemoryView and owner-containment search boundaries
- Source registration, trust metadata, and source write/use permissions
- RawEvidence ingestion and lineage
- Policy and `requested_use` filtering
- search, atom feed, context, ask, feedback, job status, and delete propagation contracts

Applications own:

- app UI and interaction flow
- app-local sessions, messages, threads, files, projects, folders, or channels
- batching and formatting before ingestion
- deciding when to write memory-worthy input
- deciding how returned Memory results are presented or combined with app-local context

App-local resources are bindings or projections over Memory-owned principals and resources. They are not new canonical Memory identity, membership, or space concepts.

## Fixed integration surface

Production app integrations use:

- `Authorization: Bearer <mem_app_live_...>` for app service credential authentication
- `X-Api-Key: <mem_app_live_...>` as the Memory credential transport when infrastructure already uses `Authorization`
- `X-Memory-On-Behalf-Of-Type` and `X-Memory-On-Behalf-Of-Id` for delegated actor context
- `X-Memory-App-Binding-Id` after an app binding exists
- `POST /v1/app-bindings/bootstrap` for idempotent app-local resource binding
- `read`, `write`, `delete`, `export`, `admin` as the minimum Memory resource permissions

AppCredential scopes authorize API families. They do not replace MemorySpace or MemoryView membership checks for the delegated principal.

## Space and view model

MemorySpace is the canonical collaboration and retrieval boundary. Applications may present a MemorySpace under an app-local name:

- Chat presents a MemorySpace as a channel, DM, or personal-agent DM.
- Another app may present a MemorySpace as a project, folder, case, repository, workspace, or other local object.

The app-local object is a binding to a MemorySpace. It must not become a second canonical space model.

MemoryView is the explicit cross-space retrieval boundary. Owner-containment scope is a MemoryView-equivalent runtime boundary derived from Memory-owned principal containment. Apps must not pass arbitrary space lists or rely on app-local filtering after Memory has already returned cross-space evidence.

## Core API families

App integrations use these Memory API families:

```text
POST /v1/app-bindings/bootstrap
POST /v1/memory-spaces/{space_id}/ingestions
GET  /v1/jobs/{job_id}

POST /v1/memory-spaces/{space_id}/search
POST /v1/memory-spaces/{space_id}/atoms/feed
POST /v1/memory-spaces/{space_id}/context
POST /v1/memory-spaces/{space_id}/ask

POST /v1/memory-views/{view_id}/search
POST /v1/memory-views/{view_id}/atoms/feed
POST /v1/memory-views/{view_id}/context
POST /v1/memory-views/{view_id}/ask

POST /v1/memory-scopes/owner-containment/search
POST /v1/memory-scopes/owner-containment/atoms/feed
POST /v1/memory-scopes/owner-containment/context
POST /v1/memory-scopes/owner-containment/ask

POST /v1/memory-spaces/{space_id}/feedback
POST /v1/memory-spaces/{space_id}/delete-propagations
```

Use `search` for ranked matching memory records, `atoms/feed` for active atom snapshot sync ordered by creation time, `context` for Memory-produced context text plus evidence/provenance, and `ask` for normal user-facing grounded natural-language answers. A `context` response always includes both `context_text` and bounded structured `evidence`; apps do not request compact-only or full-pack variants.

Every read must include `requested_use`. Memory must not return memory that is unauthorized for the delegated principal or blocked by policy for the requested use.

## Principal and membership model

Memory owns canonical principal and membership state so memories can be reused safely across applications. Read [Principals and Membership Contract](principals-and-membership.md) before implementing app user provisioning, app binding bootstrap, membership sync, or cross-space retrieval.

## App-specific extensions

An app may define app-local Source types and metadata conventions, such as Chat's `chat_conversation_segment`. These identify provenance and traceability. They must not cause Memory to branch structurally by app name.

New app contracts belong under `apps/<app_id>/`. Keep app-specific batching, UI, message lifecycle, and local storage behavior there.
