# Memory App Integration Guide

This guide is the practical call sequence for an application backend integrating with Memory. It complements the core contract in `README.md` and the app-specific contracts under `../apps/`.

## 1. Configure the app backend

Applications should keep Memory connection settings in backend-only configuration:

```sh
MEMORY_API_BASE_URL="https://memory.example.com"
MEMORY_APP_CREDENTIAL="mem_app_live_example.secret"
MEMORY_APP_ID="chat"
```

Current Memory production endpoint for the GCP deployment is:

```sh
MEMORY_API_BASE_URL="https://memory-api-uajejfqfxa-an.a.run.app"
```

Do not expose `MEMORY_APP_CREDENTIAL` to browsers, mobile clients, or client-side JavaScript. App clients should call their own app backend, and the app backend calls Memory.

For the normal Memory API deployment, send the app credential as the HTTP bearer token:

```http
Authorization: Bearer mem_app_live_example.secret
```

If Memory is deployed behind infrastructure that already uses the `Authorization` header, such as an IAM-protected Cloud Run service, use `Authorization` for the infrastructure token and send the Memory credential in `X-Api-Key`:

```http
Authorization: Bearer <cloud_run_identity_token>
X-Api-Key: mem_app_live_example.secret
```

For the current Cloud Run production deployment, Chat should call the production base URL above from its backend, set `Authorization` to a Cloud Run identity token for that audience, and set `X-Api-Key` to the Memory app credential stored in Chat's server-side secret manager.

Memory treats these two Memory credential transports equivalently:

- `Authorization: Bearer <mem_app_live_...>`
- `X-Api-Key: <mem_app_live_...>`

## 2. Always send the delegated actor

Production app integrations authenticate as the application and delegate each user, team, or app-local binding request to a canonical Memory principal.

Every resource read/write request made on behalf of a principal should include delegated actor headers. For example:

```http
X-Memory-On-Behalf-Of-Type: user
X-Memory-On-Behalf-Of-Id: user_001
X-Memory-App-Binding-Id: bind_chat_channel_ch_project_planning
```

`X-Memory-App-Binding-Id` is absent during first bootstrap because the binding does not exist yet. After bootstrap, app-bound MemorySpace operations must send it so Memory can audit and authorize the request in app-binding context.

The app credential scopes allow the app to call API families. They do not replace MemorySpace or MemoryView membership checks for the delegated principal.

## 3. Bootstrap the app resource once

Before ingestion or retrieval, bind the app-local resource to a canonical MemorySpace:

```http
POST /v1/app-bindings/bootstrap
```

Use a stable idempotency key derived from the app-local resource:

```http
Idempotency-Key: chat:channel:ch_project_planning
```

For the current Chat example, use the payload shape shown in `../apps/chat/examples/chat-bootstrap.json`.

App-specific connectors may pass a full `source` definition instead of a Chat `source_template`. This is the standard path for new non-Chat connectors such as Google Workspace:

```json
{
  "source": {
    "id": "src_google_gmail_messages",
    "source_type": "google_gmail_message",
    "display_name": "Gmail messages",
    "trust_zone": 2,
    "source_authority": "mixed_email_correspondence",
    "allowed_memory_writes": ["observation", "decision", "open_loop", "preference_evidence"],
    "disallowed_memory_writes": ["user_profile", "user_preference", "policy", "procedure"],
    "default_allowed_uses": ["search", "answer_generation", "reflection"],
    "default_disallowed_uses": ["training", "external_share", "auto_action", "profile_update"],
    "default_sensitivity": "normal",
    "retention_policy": "default"
  }
}
```

The bootstrap body must include exactly one of `source_template` or `source`. Existing Chat `source_template` payloads remain valid. Generic `source` definitions must use `allowed_use` and `memory_write` labels registered by the target MemoryPack.

The app backend should persist the resolved response fields it needs for later calls:

- app binding id
- MemorySpace id
- Source id
- delegated principal memberships or permission state, if returned
- app-local resource id and MemorySpace mapping

Do not create a separate app-local canonical memory space. The app-local channel, DM, project, folder, or workspace is a binding to the Memory-owned MemorySpace.

## 4. Ingest memory-worthy input

Write through the generic ingestion endpoint:

```http
POST /v1/memory-spaces/{space_id}/ingestions
```

Use the Source returned or resolved by bootstrap. For Chat conversation batches, use:

```json
{
  "source_id": "src_chat_segments",
  "source_type": "chat_conversation_segment",
  "content": {
    "type": "text",
    "text": "..."
  },
  "metadata": {
    "app_id": "chat",
    "memory_space_id": "space_chat_channel_ch_project_planning",
    "conversation_segment_id": "chatseg_20260621_001"
  },
  "allowed_uses": ["search", "answer_generation", "reflection", "project_context"],
  "disallowed_uses": [],
  "processing": {
    "mode": "async",
    "pipeline": "default"
  }
}
```

Ingest conversation segments or curated memory inputs, not every raw app event by default. Keep source identifiers, segment ids, message ids, checksums, event time, and transform metadata so Memory can trace RawEvidence back to the app source.

For asynchronous ingestion, Memory returns a `job_id`.

## 5. Poll the ingestion job

Poll the job endpoint until the job reaches a terminal state:

```http
GET /v1/jobs/{job_id}
```

Expected statuses:

- `queued`
- `running`
- `succeeded`
- `failed`
- `dead_letter`

Treat `succeeded` as Memory acceptance of the RawEvidence and derived write result. If the app locks or freezes included source messages after Memory acceptance, do it after `succeeded`, not immediately after enqueue.

On `failed`, the app may retry according to app policy. On `dead_letter`, surface an operational alert or remediation task rather than silently dropping the segment.

## 6. Read through search, atom feed, context, or ask

For a single MemorySpace:

```http
POST /v1/memory-spaces/{space_id}/search
POST /v1/memory-spaces/{space_id}/atoms/feed
POST /v1/memory-spaces/{space_id}/context
POST /v1/memory-spaces/{space_id}/ask
```

For a MemoryView:

```http
POST /v1/memory-views/{view_id}/search
POST /v1/memory-views/{view_id}/atoms/feed
POST /v1/memory-views/{view_id}/context
POST /v1/memory-views/{view_id}/ask
```

For owner-containment scope:

```http
POST /v1/memory-scopes/owner-containment/search
POST /v1/memory-scopes/owner-containment/atoms/feed
POST /v1/memory-scopes/owner-containment/context
POST /v1/memory-scopes/owner-containment/ask
```

Choose the API by desired abstraction:

- `search`: return matching memory records for UI cards, inspection, manual selection, or app-specific logic.
- `atoms/feed`: return active, currently policy-allowed MemoryAtoms ordered by creation time for snapshot sync. This is not a mutation feed and must not be used to infer deletes, policy changes, or historical event order.
- `context`: return a ContextPack response when the app wants evidence, provenance, and retrieval trace for its own answerer or tools. The response may include `context_text`, a Memory-produced compact string for prompt input, instead of or alongside structured evidence collections.
- `ask`: return a grounded natural-language answer generated by Memory. Use this for normal user-facing memory questions.

Every read must include `requested_use`:

```json
{
  "query": "What should the assistant remember before scheduling the project planning session?",
  "requested_use": "answer_generation",
  "limit": 8
}
```

Atom feed request bodies omit `query`:

```json
{
  "requested_use": "search",
  "limit": 100,
  "cursor": null
}
```

Owner-containment atom feed additionally includes `request_origin`. Cursor values are opaque and scope-bound to the Memory resource and resolved MemorySpace set; pass the returned `next_cursor` back unchanged to the same endpoint/scope while `has_more` is true.

Memory must not return memory that is unauthorized for the delegated principal or blocked by policy for the requested use.

## 7. Handle deletes and redactions through Memory

Applications must not delete only their local binding and assume Memory is deleted. Use Memory delete propagation APIs for memory deletion or redaction workflows:

```http
POST /v1/memory-spaces/{space_id}/delete-propagations
```

The app should keep enough source metadata to request deletion by the Memory target ids and, when applicable, by app-local source identifiers recorded in ingestion metadata.

## 8. Minimal backend request helper

The app backend should centralize Memory requests so all calls consistently attach credentials and delegated actor headers:

```ts
type MemoryOwnerType = "user" | "app" | "team" | "organization" | "system";

export async function callMemory(input: {
  method: "GET" | "POST";
  path: string;
  body?: unknown;
  delegated: { type: MemoryOwnerType; id: string };
  appBindingId?: string;
}) {
  const headers: Record<string, string> = {
    Authorization: `Bearer ${process.env.MEMORY_APP_CREDENTIAL}`,
    "Content-Type": "application/json",
    "X-Memory-On-Behalf-Of-Type": input.delegated.type,
    "X-Memory-On-Behalf-Of-Id": input.delegated.id
  };

  if (input.appBindingId) {
    headers["X-Memory-App-Binding-Id"] = input.appBindingId;
  }

  const response = await fetch(`${process.env.MEMORY_API_BASE_URL}${input.path}`, {
    method: input.method,
    headers,
    body: input.body === undefined ? undefined : JSON.stringify(input.body)
  });

  const payload = await response.json().catch(() => null);
  if (!response.ok) {
    throw new Error(`Memory ${input.method} ${input.path} failed: ${response.status} ${JSON.stringify(payload)}`);
  }
  return payload;
}
```

For IAM-protected Cloud Run, replace the helper's Memory credential header with `X-Api-Key` and set `Authorization` to the Cloud Run identity token.

## 9. Integration checklist

Before enabling production traffic:

- Memory app credential is stored server-side only.
- The app sends delegated actor headers on user/team resource calls.
- The app does not use unresolved local users as delegated Memory principals.
- Bootstrap is idempotent and the app persists the returned binding.
- App-bound requests include `X-Memory-App-Binding-Id` after bootstrap succeeds.
- Ingestion includes `content.type: "text"` and source metadata.
- Membership add, removal, deactivation, auto-join, and role changes are durably synchronized to Memory.
- Async jobs are polled to `succeeded` before local source messages are locked.
- Reads always include `requested_use`.
- Cross-space reads use MemoryView or owner-containment APIs, never direct database access.
- Deletes and redactions use Memory APIs rather than app-local deletion only.
