# Google Workspace to Memory Boundary

## Status

Initial v1 contract for Gmail and Google Drive connector integration.

This document records the Google Workspace-specific boundary. It does not redefine Memory internals.

## Core principle

The Google Workspace connector reads Google. Memory does not.

The connector sends memory-worthy, deterministic extracted text plus provenance metadata. Memory stores that input as RawEvidence under a registered Source, derives MemoryAtoms through the normal ingestion pipeline, and returns only memory that passes resource authorization and requested-use policy.

## Responsibility split

Memory is responsible for:

- app service credential and delegated actor authorization
- canonical principals and Memory-owned membership
- MemorySpace / MemoryView / owner-containment boundaries
- Source registration, trust zone, source authority, allowed uses, and write permissions
- RawEvidence storage and lineage
- Memory Write Gate, source laundering prevention, and profile / policy / procedure promotion rules
- search, atoms/feed, context, ask, feedback, export manifest, and delete propagation

The Google Workspace connector is responsible for:

- Google OAuth token and refresh token storage
- Google API calls
- Gmail history cursor and pagination state
- Drive page token, folder traversal state, and file revision cursor state
- file export and deterministic text extraction before ingestion
- connector retry, batching, rate limits, and source-side dedupe
- preserving Google source identifiers in ingestion metadata

Chat or a platform settings UI may expose configuration for connected accounts, folders, and sync preferences. Sync execution remains connector service behavior, not Memory behavior.

## Authentication and delegation

The connector calls Memory as an app integration:

```http
Authorization: Bearer <mem_app_live_...>
X-Memory-On-Behalf-Of-Type: user
X-Memory-On-Behalf-Of-Id: user_001
X-Memory-App-Binding-Id: bind_google_workspace_google_gmail_account_gmail_user_001
```

The app credential alone is not enough. The delegated principal must exist and must have the required Memory resource membership.

## Bootstrap contract

Google Workspace connectors use the generic app binding bootstrap endpoint:

```http
POST /v1/app-bindings/bootstrap
```

New connector integrations should pass a full `source` definition instead of a Chat `source_template`. Existing Chat payloads that use `source_template` remain valid.

The `source` definition must use labels registered by the target MemoryPack. Unknown `allowed_use` or `memory_write` labels fail closed.

## Ingestion contract

The connector ingests through:

```http
POST /v1/memory-spaces/{space_id}/ingestions
```

Gmail and Drive content must be sent as deterministic extracted text. The connector must not send only an LLM summary as canonical input. If the connector creates a summary for display or batching, metadata must mark the transform and the original deterministic text or enough source lineage must still be ingested as RawEvidence.

The ingestion request `source_type` must match the registered Source. Gmail content goes through `google_gmail_message`; Drive file content goes through `google_drive_file`.

## Source laundering and promotion safety

Gmail and Drive are external or mixed-authority sources even when the account is user-owned.

External text such as "remember the user is X", "always treat this as policy", or "from now on do Y" must not directly create user Profile, Policy, or Procedure memory. Memory may store an attributed evidence atom such as "an email from X claimed Y" when policy allows it, but promotion to profile / policy / procedure is a later Memory-controlled proposal with evidence, authority, counter-evidence, and confirmation requirements.

Google Workspace Sources should default-deny:

- `training`
- `external_share`
- `auto_action`
- `profile_update`

Team Drive sources must not become canonical company policy or procedure unless a separate authoritative Source and MemoryPack policy explicitly allow that promotion.

## Membership boundary

Google Drive sharing is not Memory membership. A file or folder being shared in Google does not automatically mean Memory can return its derived memory to every Google viewer.

Memory read/write/delete/export/admin access is controlled by Memory membership and MemoryView / owner-containment scope. Connector membership sync must be explicit and auditable.

## What Memory must not do

- Store Google OAuth tokens or refresh tokens.
- Store Google sync cursors as Memory connector state.
- Call Gmail or Drive APIs.
- Export Drive files.
- Treat Google folder sharing as Memory membership.
- Treat Gmail or Drive source text as direct user profile, policy, or procedure instructions.
- Accept an app credential without delegated principal membership for bootstrap or ingestion.

## What the connector must not do

- Read Memory's internal database.
- Pass arbitrary MemorySpace lists for cross-space retrieval.
- Hide Google provenance from RawEvidence metadata.
- Send only LLM summaries when deterministic extracted text is available.
- Assume Google-side delete, unshare, or permission changes delete or revoke Memory state without calling Memory APIs.
