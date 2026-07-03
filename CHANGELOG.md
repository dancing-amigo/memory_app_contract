# 変更履歴

## 2026-07-03

- `bootstrap` をアプリ resource と MemorySpace / Source の接続初期化に狭め、通常の MemorySpace membership 調整は独立した membership API と example に分離した。
- Markdown ドキュメントを日本語化した。
- `apps/` を `apps/README.md` のみへ整理した。Memory が知るべきアプリ情報は、アプリ一覧、入力 Source、API が呼ばれる場面、利用パターンだけに限定する。
- Chat 固有の ingestion request example を削除した。ingestion API の request / response example は `memory/examples/ingestion.json` を正本とする。
- 契約 repo を root `README.md`、`memory/README.md`、汎用 `memory/examples/*.json`、`apps/README.md` に整理した。
- `context` response は常に `context_text` と structured `evidence` の両方を返す契約にした。外部契約では pack id を露出せず、API response は `context_id` を使う。

## 2026-06-28

- single MemorySpace、MemoryView、owner-containment scope 向けの atoms/feed read endpoint を追加した。feed は active かつ policy-allowed な MemoryAtom の作成順 snapshot であり、mutation feed ではない。
- Chat conversation segment の default idle close window を 72 時間から 24 時間へ変更した。
- bounded thread context 用の Chat ingestion metadata を追加した。

## 2026-06-26

- 初期 production boundary では first-class app registration / installation table を必須にしない方針を明確化した。active app service credential、active app binding、active delegated principal、Memory-owned resource membership で fail-closed authorization を行う。

## 2026-06-25

- `X-Api-Key` を、infrastructure が `Authorization` を使う場合の Memory credential transport として文書化した。
- app-bound operation では bootstrap 後に `X-Memory-App-Binding-Id` を送ることを明確化した。
- production Chat user は canonical Memory principal として provision されるまで memory-capable にしてはいけないことを明確化した。
- Chat channel / DM の membership 変更を Memory membership に同期する必要があることを明確化した。
- owner-containment retrieval でも app-bound channel / DM Space は Memory-owned read membership で filter される必要があることを明確化した。

## 2026-06-23

- owner-containment `search` / `context` / `ask` scope を追加した。
- MemoryView `search` / `context` / `ask` endpoint を initial runtime endpoint として扱うことを明確化した。
- Chat Ask API 契約を追加した。
- app integration HTTP contract として delegated actor headers、`mem_app_live_` AppCredential、`POST /v1/app-bindings/bootstrap`、membership permissions、Chat Source template を整理した。

## 2026-06-21

- 共有 Memory application contract repo を作成した。
- Chat と Memory の初期 boundary contract を追加した。
- participating repository の submodule path を root-level `memory_app_contract/` に標準化した。
- Chat production integration では app service credential と delegated `on_behalf_of` semantics を使う方針を追加した。
