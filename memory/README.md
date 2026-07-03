# Memory API 契約

この文書は、アプリ backend が Memory API を使うための契約です。どの request を送るか、どの header が必要か、何が返ると期待できるかをここに集約します。

## 認証と delegated principal

アプリ backend は app service credential で Memory を呼びます。

```http
Authorization: Bearer mem_app_live_example.secret
```

Cloud Run などで `Authorization` を infrastructure token に使う場合は、Memory credential を `X-Api-Key` に入れます。

```http
Authorization: Bearer <infrastructure_token>
X-Api-Key: mem_app_live_example.secret
```

ユーザー、team、app-bound resource の read / write では、必ず Memory の delegated principal を送ります。

```http
X-Memory-On-Behalf-Of-Type: user
X-Memory-On-Behalf-Of-Id: user_001
X-Memory-App-Binding-Id: bind_example_project_001
```

`X-Memory-App-Binding-Id` は初回 bootstrap 時にはまだ存在しないので不要です。binding 作成後の app-bound request では送ります。

App credential の scope は API family を呼べるかを決めるだけです。対象 MemorySpace / MemoryView を読めるか、書けるかは delegated principal の membership で判定します。

## 基本 object

- `MemorySpace`: 保存と collaboration の canonical boundary。
- `MemoryView`: cross-space retrieval の明示 boundary。
- `Source`: 入力元の provenance、trust、write/use default。
- `RawEvidence`: Memory に入った原本入力。
- `MemoryAtom`: RawEvidence から派生した evidence。
- `context`: Memory が作る compact な context string と bounded structured evidence。

アプリは MemorySpace を channel、folder、project、DM、case などとして見せてもよいですが、それらは MemorySpace への binding であり、別の canonical space model ではありません。

## 権限

最小 permission は次です。

- `read`: search、atoms/feed、context、ask、job status、non-export read
- `write`: ingestion、feedback
- `delete`: delete propagation、redaction
- `export`: export API
- `admin`: Source、membership、binding、policy、MemorySpace、MemoryView 設定

`admin` は content の read / write / delete / export を自動的には含みません。

## API

```text
POST /v1/app-bindings/bootstrap
POST /v1/memory-spaces/{space_id}/memberships
PUT  /v1/memory-spaces/{space_id}/memberships
DELETE /v1/memory-spaces/{space_id}/memberships/{principal_type}/{principal_id}
POST /v1/principal-memberships
DELETE /v1/principal-memberships/{member_type}/{member_id}/{group_type}/{group_id}

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

## Bootstrap

アプリローカル resource を canonical MemorySpace と Source に binding します。

例: [bootstrap.json](examples/bootstrap.json)

request は app credential と `Idempotency-Key` に対して idempotent です。同じ内容の再送は同じ resource を返し、矛盾する内容の再送は idempotency conflict になります。

Bootstrap はアプリ接続の初期化 API です。通常の membership 追加、削除、権限変更、同期、reconcile には使いません。

Bootstrap は次を作成または解決します。

- app binding
- MemorySpace
- Source

新しい MemorySpace を作る場合だけ、request は `initial_memberships` を含められます。これは最初の admin / writer などを seed するための escape hatch であり、既存 Space の membership 調整ではありません。既存 Space や既存 binding に対する再 bootstrap は membership を変更してはいけません。

## Membership

MemorySpace の membership を追加、同期、削除します。

例: [membership.json](examples/membership.json)

アプリ側の channel、DM、project、folder などで参加者や権限が変わった場合、アプリは bootstrap を再利用せず、membership API を呼びます。

```text
POST   /v1/memory-spaces/{space_id}/memberships
PUT    /v1/memory-spaces/{space_id}/memberships
DELETE /v1/memory-spaces/{space_id}/memberships/{principal_type}/{principal_id}
```

`POST` は単一 principal の membership を作成または再有効化します。`PUT` は対象 MemorySpace の membership set を request body に合わせて reconcile します。`DELETE` は単一 principal の membership を無効化します。

principal containment は、user が team に属する、team が organization に属する、などの canonical principal 関係を Memory に登録するための API です。これは MemorySpace membership とは別物です。

```text
POST   /v1/principal-memberships
DELETE /v1/principal-memberships/{member_type}/{member_id}/{group_type}/{group_id}
```

## Ingestion

memory-worthy な text を RawEvidence として投入します。

例: [ingestion.json](examples/ingestion.json)

アプリはローカル event をそのまま細かく送るのではなく、アプリ側で coherent な text segment にまとめてから送ります。deterministic な source text がある場合、LLM summary だけを canonical input として送ってはいけません。

async ingestion は `job_id` を返します。`GET /v1/jobs/{job_id}` を `succeeded`、`failed`、`dead_letter` まで poll します。

## Read API

すべての read は `requested_use` を含みます。Memory は delegated principal が access できない記憶、または policy が requested use に許可しない記憶を返してはいけません。

### search

UI card、inspection、manual selection、app-specific logic のために ranked MemoryAtom hits を返します。

例: [search.json](examples/search.json)

### atoms/feed

現在 active で、requested use policy を通過した MemoryAtom を作成順に返します。これは snapshot sync 用であり、mutation feed ではありません。delete、policy 変更、historical event order を推論するために使ってはいけません。

例: [atoms-feed.json](examples/atoms-feed.json)

### context

アプリ側の answerer、tool、action、evidence view、debug flow に渡すための reusable Memory context を返します。

例: [context.json](examples/context.json)

response は常に両方を含みます。

- `context_text`: prompt input に使う compact な Memory-produced string。
- `evidence`: `context_text` を作るために使った bounded structured evidence。可能な場合は MemoryAtom と RawEvidence record / excerpt を含みます。

request-time の response format selector はありません。Memory は evidence limit を所有し、常に返る response が app で使えるサイズになるよう調整します。

`context_text` は lossy で prompt-ready な surface です。structured evidence、source refs、audit fields の代替として parse してはいけません。

### ask

Memory が直接 grounded natural-language answer を生成して返します。

例: [ask.json](examples/ask.json)

アプリが「Memory にこの質問へ答えてほしい」場合は `ask` を使います。アプリ自身の prompt、tool、action flow に Memory context を渡したい場合は `context` を使います。

## Cross-Space Read

cross-space read は MemoryView または owner-containment scope 経由で行います。アプリが任意の space list を渡したり、Memory が返した後で unauthorized evidence をアプリ側 filter したりしてはいけません。

Owner-containment は `request_origin` から解決します。

```json
{ "type": "user", "id": "user_001" }
```

user origin は user 自身の MemorySpace と、それを包含する team / organization Space を含められます。team origin は team と containing organization Space を含められますが、member personal Space は default では含みません。

## Delete / Redaction

アプリローカルの binding だけを削除しても、Memory data は削除されたことになりません。削除や redaction は次を使います。

```http
POST /v1/memory-spaces/{space_id}/delete-propagations
```

アプリは削除・redaction request に必要な source id、raw evidence id、app-local source id、checksum を保持します。
