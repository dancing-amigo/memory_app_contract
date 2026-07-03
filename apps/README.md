# アプリ一覧と利用パターン

この文書は、Memory が知っておくとよいアプリ側の情報だけを記録します。

Memory はアプリごとの内部 model を知る必要はありません。Memory にとって重要なのは、どの Source から入力が来るのか、どの場面でどの API が呼ばれるのか、返した context / answer がどう使われるのかです。

## 記録すること

各アプリについて、必要なら次だけを書きます。

- アプリの目的
- Memory に登録される代表的な Source
- `bootstrap` が呼ばれる場面
- `membership` が呼ばれる場面
- `ingest` が呼ばれる場面
- `context` が呼ばれる場面
- `ask` が呼ばれる場面
- Memory response がアプリ側でどう使われるか
- Memory がそのアプリ向けに構造分岐してはいけない点

API request / response の正本 example は `memory/examples/*.json` に置きます。アプリ別の API request example は原則置きません。

## Chat

目的:

- ユーザーや team の会話を Memory に蓄積し、後続の会話、tool、action、検索、回答に使う。

代表的な Source:

- `source_id`: `src_chat_segments`
- `source_type`: `chat_conversation_segment`

入力:

- Chat は会話を app 側で segment 化し、Memory の generic ingestion API に text と metadata を送る。
- Memory は Chat の message model を知る必要はない。
- Memory にとって必要なのは、Source、RawEvidence text、app metadata、lineage / delete / audit に使える app-local identifier だけ。

`bootstrap` が呼ばれる場面:

- channel、DM、personal-agent DM など、Chat 側 resource を MemorySpace / Source に binding する時。

`membership` が呼ばれる場面:

- channel、DM、project などに対応する owner principal の参加者が変わった時。
- Chat 側 resource の参加者一覧を principal membership に同期する時。

`ingest` が呼ばれる場面:

- Chat が memory-worthy な conversation segment を確定した時。
- segment の切り方、未送信 message の扱い、UI 上の lock は Chat 側の責務。

`context` が呼ばれる場面:

- Chat が自前の prompt、tool、action、workflow に Memory context を混ぜたい時。
- Chat は `context_text` を prompt input として使い、`evidence` を citation、debug、tool grounding、audit に使う。

`ask` が呼ばれる場面:

- Chat が Memory に直接 user-facing answer を生成してほしい時。

Memory がしないこと:

- Chat の DB や message schema を読む。
- Chat の channel / DM / thread を canonical Memory object として扱う。
- `app_id = chat` だけを理由に extraction、policy、retrieval を構造分岐する。
- Chat 側 membership filter を信頼して unauthorized evidence を返す。
