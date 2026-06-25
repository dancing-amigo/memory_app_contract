# Principals and Membership Contract

This document explains why Memory owns canonical principals and memberships, what Memory must provide to applications, and what applications may rely on without reading Memory implementation code.

## Why Memory owns this

Memory is the shared memory system behind Chat and future applications. A user may create useful memory through Chat and later ask for that memory through another application. That only works if the user, team, organization, app, and membership graph are canonical in Memory rather than being defined separately by each application.

If Chat creates a local user or local channel membership and that change never reaches Memory, another application cannot safely answer these questions:

- whether the same human is the same Memory user
- which MemorySpaces the user can read, write, delete, export, or administer
- which team or organization memories should be in scope for the user
- whether a removed channel member still has access through stale Memory membership

For that reason, application-local users, teams, channels, projects, folders, and memberships are projections or bindings. Memory is the authority for canonical principal existence, resource membership, principal containment, and cross-application retrieval authorization.

## Principal model

A Memory principal is the canonical actor or group identity used by Memory authorization and retrieval scope. The current principal types are:

- `user`
- `app`
- `team`
- `organization`
- `system`

An application may store local user, team, or workspace records, but it must not treat those records as Memory principals until Memory has created or resolved the corresponding principal.

Production applications must not issue delegated Memory requests for a missing, inactive, or unresolved Memory principal.

## Application user states

Applications should distinguish local account state from Memory principal state.

Recommended state model:

```text
local_pending
  -> memory_principal_resolved
  -> memory_capable
```

`local_pending` means the application may have an invite, login attempt, or local account row, but Memory has not confirmed a canonical principal. The app must not write memory, create memory-bound resources, or issue delegated Memory reads for that user.

`memory_principal_resolved` means Memory has confirmed the canonical principal id and status. The app may create or refresh app-local projections that reference that principal.

`memory_capable` means the app can use the principal in delegated Memory requests, subject to resource membership and policy.

If Memory principal provisioning is unavailable, the app should fail closed for memory-capable workflows and either keep the local user pending or run a clearly degraded non-memory mode.

## Resource membership

Resource membership grants a principal permissions on a Memory resource such as a MemorySpace or MemoryView.

Minimum permissions are independent:

- `read`: search, context, ask, view read, job status, and non-export read APIs
- `write`: ingestion and feedback
- `delete`: delete propagation and redaction requests
- `export`: export APIs
- `admin`: Source, membership, app binding, policy, MemorySpace, and MemoryView configuration

`admin` does not imply `read`, `write`, `delete`, or `export`. A user who needs full owner-like access should receive each required permission explicitly, for example `["read", "write", "delete", "export", "admin"]`.

Memory must evaluate resource membership using the delegated principal from `X-Memory-On-Behalf-Of-Type` and `X-Memory-On-Behalf-Of-Id`. The app credential authenticates the calling app; it is not resource access by itself.

## Principal containment

Principal containment describes membership between principals, for example:

```text
user_001 -> team_001 -> organization_001
```

Memory uses principal containment to resolve owner-containment query scope. Principal containment does not automatically grant `write`, `delete`, `export`, or `admin` on app-bound collaboration spaces.

For app-bound or member-scoped MemorySpaces, owner-containment must work in two stages:

1. resolve candidate spaces from owner containment
2. filter candidate spaces by Memory-owned resource `read` membership for the delegated principal

A broad team or organization owner is not enough to include every private channel, group DM, or DM MemorySpace in a user's cross-space result.

## App binding bootstrap

Applications bind local resources to Memory resources through:

```http
POST /v1/app-bindings/bootstrap
```

Bootstrap is for idempotently resolving or creating an app binding, MemorySpace, Source, and initial memberships. It is not a complete lifecycle API for every later membership transition unless the request is explicitly defined as a full active membership snapshot.

Bootstrap requirements:

- It is idempotent by app id plus `Idempotency-Key`.
- The delegated principal must be included in the requested initial memberships.
- First creation may create the delegated principal's initial resource membership.
- Existing resource changes require appropriate `admin` membership unless a narrower API says otherwise.
- Reusing an idempotency key with an incompatible body must fail with an idempotency conflict.

## Membership synchronization

Applications must propagate every membership transition that affects Memory access to Memory. This is structural authorization state, not content ingestion state.

The app must synchronize:

- resource creation that grants initial members access
- member add
- member remove or leave
- role or permission changes
- user deactivation
- automatic join to an open resource
- app-local policy changes that affect who may read, write, delete, export, or administer memory

Content memory being off does not disable membership synchronization. A channel or project with message ingestion disabled still needs Memory membership state so future reads, writes, deletion, export, and cross-space retrieval are authorized correctly.

## Membership removal

Additive bootstrap alone is not valid removal semantics. Sending a shorter `memberships` array to an additive bootstrap endpoint must not be assumed to revoke omitted principals.

Production Memory must expose one of these semantics before app membership removal is production-safe:

- an explicit revoke/deactivate API for MemorySpace and MemoryView memberships
- an explicit full membership reconciliation API
- an explicit bootstrap mode that is documented as a full active snapshot and deactivates omitted principals

Full reconciliation requests must be marked unambiguously, for example with an endpoint, mode field, or endpoint semantics that cannot be confused with additive bootstrap. Memory must audit create, update, revoke, deactivate, and full-reconcile operations.

Stale active membership after app-local removal is an authorization bug.

## Required Memory behavior

Memory must provide or define:

- canonical principal create/resolve behavior for users, apps, teams, organizations, and system actors
- principal status sufficient to reject delegated requests for missing or inactive principals
- resource membership create/update/revoke or full-reconcile behavior
- principal containment create/update/deactivate behavior
- resource authorization based on app credential scope plus delegated principal membership
- owner-containment retrieval that also filters app-bound member-scoped spaces by resource read membership
- audit or mutation logs for principal, membership, app binding, and policy changes

Memory does not need a separate first-class app registration or installation table for the initial production boundary if the runtime can fail closed using:

- active app service credential status and scope
- active app binding for app-bound resource operations
- active delegated principal
- Memory-owned resource membership for the requested action

First-class app registration and installation records are optional lifecycle extensions. Add them when Memory needs app-wide disable/suspend, tenant or workspace uninstall, central capability management, or installation audit independent of credentials and bindings. They must not replace delegated principal membership or app binding authorization.

## Required application behavior

Applications must:

- store Memory principal ids or deterministic mappings only after Memory confirms them
- keep local users pending or degraded if Memory principal resolution fails
- use app service credentials, not per-user Memory API keys
- send delegated actor headers on resource operations
- send `X-Memory-App-Binding-Id` for app-bound operations after bootstrap has created the binding
- persist app binding id, MemorySpace id, Source id, and membership sync status
- propagate every membership change to Memory through durable sync
- fail closed for Memory reads and writes when the delegated principal or resource binding is unresolved

## Remaining validation and optional lifecycle

The current Memory boundary covers canonical principal resolution, delegated app-bound authorization, explicit membership revoke/full reconcile, and owner-containment resource membership filtering. Remaining work is:

- broader owner-containment no-leak integration tests for app-bound private spaces
- first-class app registration / installation lifecycle only if product operations require app-wide or tenant-level lifecycle control beyond active credentials and active bindings

These are not reasons to move identity or membership authority into applications. They are Memory-side contract and runtime gaps that must be closed while preserving Memory as the canonical authority.
