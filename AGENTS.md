# AGENTS.md

このリポジトリは Memory System と、Chat などのアプリ repo の共有契約です。

## 必須運用ルール

この repo を読む・編集する前に実行してください。

```sh
git pull --ff-only
```

この repo を編集したら、作業終了前に commit / push してください。

```sh
git status --short
git add <changed-files>
git commit -m "<契約変更の要約>"
git push
```

push が認証や remote 設定で失敗した場合は、その理由を明示し、local commit までは作ってください。

## 契約の責務

Memory repo は Memory API、MemorySpace、MemoryView、Source、Policy、membership、lifecycle semantics を所有します。

アプリ repo は UI、app-local session / message model、batching、formatting、Memory context の使い方を所有します。

この repo は Memory repo の ADR や product docs を置き換えません。アプリが Memory と連携する前に読む cross-repository contract を記録します。

## 編集ルール

- Memory internals をここで再定義しない。
- Memory が app name ごとに構造分岐する必要がある app-specific behavior を追加しない。
- unresolved integration question は、現在の契約に影響する場合だけ `memory/README.md` または `apps/README.md` に短く残す。それ以外はこの repo 外で管理する。
- example は現在の Memory API と揃える。
- 契約変更が Memory API、data model、policy、evaluation、membership の変更を必要とする場合は、同じ作業で Memory repo の docs または ADR も更新する。
