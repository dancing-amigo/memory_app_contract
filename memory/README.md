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

App credential の scope は API family を呼べるかを決めるだけです。対象 MemorySpace の read / write、owner-containment で解決された MemorySpace の read は MemorySpace の owner principal と delegated principal の principal membership で判定します。

## 基本 object

- `MemorySpace`: 保存と collaboration の canonical boundary。必ず 1 つの owner principal に対応します。
- `Source`: 入力元の provenance、trust、write default。
- `RawEvidence`: Memory に入った原本入力。
- `MemoryAtom`: RawEvidence から派生した evidence。
- `context`: アプリ側の prompt、tool、action、evidence view、debug flow に渡すための bounded structured evidence。

アプリは MemorySpace を channel、folder、project、DM、case などとして見せてもよいですが、それらは MemorySpace への binding であり、別の canonical space model ではありません。複数 user の集まりに対応する MemorySpace は、user を Space に直接所属させるのではなく、team / organization owner principal と principal membership で管理します。

## 権限

最小 permission は次です。

- `read`: context、ask、job status
- `write`: ingestion
- `admin`: Source、membership、binding、policy、MemorySpace 設定

`admin` は content の read / write を自動的には含みません。

## API

```text
POST /v1/app-bindings/bootstrap
DELETE /v1/app-bindings/{binding_id}
POST /v1/principal-memberships
DELETE /v1/principal-memberships/{member_type}/{member_id}/{group_type}/{group_id}

POST /v1/memory-spaces/{space_id}/ingestions
GET  /v1/jobs/{job_id}

POST /v1/memory-spaces/{space_id}/context
POST /v1/memory-spaces/{space_id}/ask
```

## Bootstrap

アプリローカル resource を canonical owner principal、MemorySpace、Source に binding します。

例: [bootstrap.json](examples/bootstrap.json)、[app-binding-delete.json](examples/app-binding-delete.json)

request は app credential と `Idempotency-Key` に対して idempotent です。同じ内容の再送は同じ resource を返し、矛盾する内容の再送は idempotency conflict になります。

Bootstrap はアプリ接続の初期化 API です。

Bootstrap は次を作成または解決します。

- owner principal
- MemorySpace。request の `memory_space.owner` はその Space に対応する owner principal です。
- Source
- app binding。app resource と owner principal / MemorySpace / Source の対応です。

bootstrap example は最小 shape だけを示します。storage、model、retention、write policy などの詳細設定は Memory repo の API / ADR 側で管理します。

Bootstrap は principal membership の追加、削除、同期、reconcile には使いません。

app binding を削除しても、owner principal、MemorySpace、Source、Memory data は削除されません。アプリローカル resource との対応だけを無効化します。

## Principal Membership

principal containment を追加、削除します。

例: [membership.json](examples/membership.json)

principal containment は、user が team に属する、team が organization に属する、などの canonical principal 関係を Memory に登録するための API です。

```text
POST   /v1/principal-memberships
DELETE /v1/principal-memberships/{member_type}/{member_id}/{group_type}/{group_id}
```

`POST` は単一 member principal を group principal に追加または再有効化します。`DELETE` は単一 member principal を group principal から無効化します。アプリ側の channel、DM、project、folder などで参加者が変わった場合、アプリは対応する owner principal の principal membership を更新します。

## Ingestion

memory-worthy な text を RawEvidence として投入します。

例: [ingestion.json](examples/ingestion.json)

アプリはローカル event をそのまま細かく送るのではなく、アプリ側で coherent な text segment にまとめてから送ります。deterministic な source text がある場合、LLM summary だけを canonical input として送ってはいけません。

async ingestion は `job_id` を返します。`GET /v1/jobs/{job_id}` を `succeeded`、`failed`、`dead_letter` まで poll します。

## Read API

Memory は delegated principal が access できない記憶を返してはいけません。

read API は request body に `include_owner_containment` を含められます。

- 省略時または `false`: path の `space_id` だけを読み取り候補にします。
- `true`: path の `space_id` の owner principal から principal containment をたどり、その owner principal と containing principal が owner の MemorySpace を読み取り候補にします。

`include_owner_containment` は `context`、`ask` だけで使います。`ingestion` は常に path の `space_id` に対する operation です。

### context

アプリ側の answerer、tool、action、evidence view、debug flow に渡すための reusable structured evidence を返します。

例: [context.json](examples/context.json)

response は bounded evidence を返します。

- `evidence`: authorization を通過し、query に対して final selected order に並べられた bounded structured evidence。MemoryAtom と RawEvidence record / excerpt を含みます。
- `budget`: Memory が見積もった evidence payload token budget。`estimated_tokens` は `target_tokens` 以下でなければなりません。

request-time の response format selector や limit はありません。Memory は evidence 量と順序を所有し、常に返る response が app で使えるサイズになるよう調整します。

`context_text` は返しません。structured evidence を prompt、tool、action、UI 用の text に組み立てる責任はアプリ側にあります。アプリは Memory の内部 DB や internal ContextPack を直接読んではいけません。

`evidence.memory_atoms[].role` は `direct` / `related` など、アプリが evidence の扱いを判断するための最低限の順序・役割情報です。debug ranking、omitted reason、policy trace は external response には含めません。

### ask

Memory が直接 grounded natural-language answer を生成して返します。

例: [ask.json](examples/ask.json)

アプリが「Memory にこの質問へ答えてほしい」場合は `ask` を使います。アプリ自身の prompt、tool、action flow に Memory context を渡したい場合は `context` を使います。

`ask` response は assembled context や context id を返しません。Memory は内部では bounded ContextPack を作って回答生成に使いますが、アプリが evidence や context を必要とする場合は `context` を明示的に呼びます。

## Cross-Space Read

cross-space read は `/v1/memory-spaces/{space_id}/{read_api}` に `include_owner_containment: true` を指定して行います。アプリが任意の space list を渡したり、Memory が返した後で unauthorized evidence をアプリ側 filter したりしてはいけません。

例: [owner-containment-context.json](examples/owner-containment-context.json)

```json
{
  "include_owner_containment": true
}
```

owner-containment は次の順で解決します。

1. path の `space_id` から MemorySpace を解決する。
2. その MemorySpace の owner principal を解決する。
3. owner principal から principal containment をたどり、owner principal と containing principal を候補 owner にする。
4. 候補 owner が owner の MemorySpace を読み取り候補にする。

user owner Space の場合は user 自身が owner の MemorySpace と、その user が属する team / organization が owner の MemorySpace を含められます。team owner Space の場合は team が owner の MemorySpace と containing organization が owner の MemorySpace を含められますが、member user が owner の personal Space は default では含みません。

MemorySpace containment relation は owner-containment の解決には使いません。アプリは sibling Space、child Space、任意の related Space を request body で追加してはいけません。

## Binding Delete

アプリローカルの binding だけを削除しても、Memory data は削除されたことになりません。binding は次で削除します。

```http
DELETE /v1/app-bindings/{binding_id}
```

Memory data の削除や redaction API は現契約には含めません。現在の Chat 連携は add-only ingestion を前提とし、source message deletion は Memory に伝播しません。
