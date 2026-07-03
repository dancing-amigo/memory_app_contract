# Memory アプリ契約

このリポジトリは、Memory System と、それを使うアプリの共有契約です。

## 読む順番

1. [Memory API 契約](memory/README.md)
2. [アプリ概念の Memory 対応](apps/README.md)

## 当面の確定状態

この契約は当面の Memory / Chat 連携の source of truth です。

Memory repo や Chat repo は、この repo の契約を先に更新せずに Memory API、principal / membership model、MemorySpace owner semantics、ingestion / context / ask の request / response shape を変更してはいけません。

## 構成

```text
memory/       アプリが Memory API を使うための契約と汎用 example。
apps/         app-local concept と Memory principal / MemorySpace / Source の対応。
AGENTS.md    この契約 repo を編集する agent 向け運用ルール。
```

## 責務

Memory が持つもの:

- canonical principal、membership、MemorySpace
- Source 登録と provenance / trust / write default
- RawEvidence ingestion
- Policy / authorization filtering
- context、ask の API 契約

アプリが持つもの:

- UI、ローカル session / message / file / project / channel
- どの入力を Memory に送るか
- 送る前の batching / formatting
- Memory から返った context や answer の使い方

アプリローカルの channel、DM、project、folder などは Memory-owned resource への binding です。canonical な Memory identity、membership、policy、space semantics をアプリ側で再定義してはいけません。

## 互換性ルール

Memory API の shape を変える場合は、まず `memory/README.md` と `memory/examples/*.json` を更新します。

アプリ固有の知識は `apps/README.md` に最小限で記録します。Memory がアプリ名ごとに構造分岐しなければならない契約は作らないでください。
