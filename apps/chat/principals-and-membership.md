# Chat Principal and Membership Contract

This document applies the Memory principal and membership contract to Chat. Read `../../memory/principals-and-membership.md` first.

## Why Chat cannot own this

Chat is a Memory-dependent application. Chat stores local users, sessions, messages, channels, DMs, and channel memberships so the chat product can work, but those records are not canonical Memory identity or authorization.

If Chat treats a local user or channel membership as final without synchronizing Memory, the same human may appear differently to another app, removed members may keep Memory access, and cross-application retrieval can leak or omit evidence. Therefore Chat must treat Memory as the source of truth for canonical principals, MemorySpace membership, principal containment, and cross-space retrieval authorization.

## Local user and Memory principal

Chat may create or hold local user state for login, invite, presence, and message authorship. A local Chat user is not memory-capable until Memory has created or resolved the corresponding canonical principal.

Chat should model user readiness separately from local account readiness:

```text
pending_local_user
  -> memory_principal_resolved
  -> memory_capable_user
```

While a user is only `pending_local_user`, Chat must not:

- write that user's messages to Memory
- create a memory-bound channel, DM, or personal-agent DM on behalf of that user
- issue delegated Memory reads or writes with that user's id
- include that user as an active MemorySpace member

If Memory principal provisioning fails, Chat may keep the local login or invite pending, or run a clearly degraded non-memory path. It must not silently invent a canonical Memory user.

## Channel and DM binding

Chat-local resources bind to MemorySpace as follows:

- `channel`: shared MemorySpace for group conversation
- `dm`: MemorySpace for the DM participants
- `personal_agent_dm`: the human user's personal MemorySpace for user-agent conversation

Chat persists app-local projections such as:

- `memory_app_binding_id`
- `memory_space_id`
- `memory_source_id`
- `memory_binding_status`
- `memory_last_error`

These fields are pointers to Memory-owned resources. They do not make Chat the owner of MemorySpace, Source, Policy, or Membership semantics.

## Bootstrap expectations

Chat uses:

```http
POST /v1/app-bindings/bootstrap
```

for first binding or idempotent refresh of a channel, DM, or personal-agent DM binding.

Chat must send:

- Chat app service credential
- delegated actor headers for a resolved Memory principal
- stable idempotency key derived from the Chat resource and operation
- app resource type and id
- target MemorySpace shape
- initial active memberships
- Source template

After bootstrap succeeds, Chat must persist the returned binding, MemorySpace, and Source ids before issuing normal app-bound reads or writes.

For app-bound Memory operations after bootstrap, Chat must send `X-Memory-App-Binding-Id`. First bootstrap is the main exception because the binding does not exist yet.

## Membership synchronization

Chat channel membership and Memory resource membership are separate records, but they must converge for production authorization.

Chat must create a durable Memory sync event for:

- channel creation
- DM creation
- personal-agent DM creation
- explicit member add
- explicit member remove or leave
- user deactivation
- open channel auto-join
- role changes that affect Memory permissions
- future visibility or scope changes that affect Memory access

Open channel auto-join is still a membership grant from Memory's point of view. It must be synchronized even if the user did not explicitly click "join".

Channel `memory_mode = off` affects message body ingestion. It must not suppress structural membership synchronization.

## Permission mapping

Chat must map product roles to explicit Memory permissions. Because `admin` does not imply content access, owner-like roles must list all permissions they require.

Recommended initial mapping:

| Chat role | Memory permissions |
| --- | --- |
| owner/admin | `read`, `write`, `delete`, `export`, `admin` |
| member/guest | `read`, `write` |
| removed/deactivated | no active membership |

If Chat intentionally withholds `export` from channel owners/admins, the product reason must be documented and tests should verify that export is denied.

## Member removal

Member removal must revoke Memory access. Chat must not rely on additive bootstrap to remove a member.

When a member is removed or leaves, Chat must do one of the following:

- call an explicit Memory membership revoke/deactivate API
- call a full membership reconciliation API with an explicitly marked active snapshot
- call a bootstrap endpoint only if the contract explicitly defines that request as a full active snapshot that deactivates omitted principals

The current Memory implementation exposes explicit membership revoke and full membership reconciliation. Retrying bootstrap with a shorter membership list is still not enough.

## Personal-agent DM

For `personal_agent_dm`, Chat binds to the human user's personal MemorySpace. The bot or agent identity is not automatically a member of the user's personal MemorySpace unless Memory policy or product requirements explicitly add it.

The delegated actor for personal-agent DM reads and writes should be the human Memory principal, not a local-only bot user.

## Cross-space retrieval

If Chat exposes "everything this user can see" retrieval, it must use MemoryView or owner-containment. Chat must not pass arbitrary space lists and then filter Memory's answer locally.

For owner-containment, Memory must filter app-bound channel, group DM, and DM spaces by Memory-owned `read` membership before returning evidence. A team or organization owner alone is not enough to expose every channel or DM space to every member.

## Failure handling

Chat should fail closed for Memory workflows when canonical identity or membership state is unresolved.

Expected handling:

- principal provisioning failed: keep user pending or non-memory-capable
- bootstrap failed: do not mark binding ready; store error and retry through durable outbox where appropriate
- membership sync failed: do not assume Memory access changed; retry and surface operational alert before production use
- member removal sync failed: treat as an authorization risk because stale active Memory membership may remain
- Memory read returns 403: do not retry as another user; surface no-access or degraded result

## What Chat can rely on

Once Memory provides the required contract behavior, Chat may rely on Memory to:

- reject delegated requests for missing or inactive principals
- enforce resource membership for MemorySpace and MemoryView operations
- enforce `requested_use` policy on returned memory
- keep app binding, MemorySpace, Source, and Membership as Memory-owned resources
- prevent owner-containment retrieval from bypassing resource membership for app-bound private spaces

## Chat implementation gaps to track

The contract expects Chat to have or add:

- a principal provisioning gate before a local user becomes memory-capable
- durable sync for every channel/DM membership transition, including open channel auto-join
- explicit member removal handling once Memory exposes revoke or full-reconcile semantics
- tests proving message memory OFF does not skip membership sync
- tests proving removed members lose Memory access after synchronization
