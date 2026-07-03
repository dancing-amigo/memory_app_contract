# アプリ概念の Memory 対応

この文書は、アプリ側のどの概念が Memory の principal、MemorySpace、Source、app binding、app_ref に対応するかだけを記録します。

API request / response の正本 example は `memory/examples/*.json` に置きます。アプリ別の利用パターンや UI flow はここでは扱いません。

## 記録すること

各アプリについて、必要なら次だけを書きます。

- app-local concept
- 対応する Memory principal
- 対応する MemorySpace owner
- 対応する Source
- principal membership の同期元
- ingestion の `app_ref`
- Memory が app-specific に解釈してはいけない点

## 共通ルール

アプリの channel、DM、folder、project、case などは Memory の canonical object ではありません。アプリ resource は app binding によって owner principal、MemorySpace、Source に紐づきます。

Memory principal は `user`、`team`、`organization` です。

- 1 人の user に対応する app resource は `user` principal を owner にします。
- 複数 user が共有する app resource は原則 `team` principal を owner にします。
- workspace、tenant、company など組織全体に対応する app resource は `organization` principal を owner にします。

MemorySpace は必ず 1 つの owner principal に対応します。複数 user の参加者は Space membership ではなく、owner principal への principal membership として同期します。

ingestion では app schema を Memory に再定義しません。Memory に渡すのは Source、canonical text、stable な `app_ref` だけです。

## Chat

Chat の会話単位は、Memory では principal と MemorySpace に分解して扱います。

| Chat concept | Memory concept | Notes |
| --- | --- | --- |
| user | `principal.type = user` | personal memory の owner。user owner の MemorySpace に membership 登録は不要。 |
| personal-agent DM / one-user DM | user principal が owner の MemorySpace | その user の個人 memory。 |
| channel | `principal.type = team` | channel 参加者を team principal の members として同期する。 |
| group DM / project room | `principal.type = team` | 複数 user が共有するなら channel と同じく team principal として扱う。 |
| workspace / tenant / company | `principal.type = organization` | org-wide memory の owner。team principal は organization に所属できる。 |
| conversation segment | RawEvidence | `source_id = src_chat_segments` で ingestion する canonical text。 |
| message / thread / reaction | direct Memory object ではない | Chat schema は Memory に再定義しない。必要な範囲だけ segment text と `app_ref` に残す。 |

代表的な Source:

- `source_id`: `src_chat_segments`
- `source_type`: `chat_conversation_segment`

Chat の app binding は、Chat resource と Memory の owner principal / MemorySpace / Source の対応です。

例:

- channel `channel_001` -> team principal `team_channel_001` -> MemorySpace `space_channel_001`
- personal-agent DM for `user_001` -> user principal `user_001` -> MemorySpace `space_user_001`
- workspace `org_001` -> organization principal `org_001` -> MemorySpace `space_org_001`

principal membership:

- channel / group DM / project room の参加者一覧を、対応する team principal の members として同期します。
- workspace 所属は user -> organization、または team -> organization の principal membership として同期します。
- user owner の personal MemorySpace のために user -> user membership は作りません。

ingestion の `app_ref`:

- `type`: `segment`
- `id`: Chat 側の stable segment id
- `occurred_at`: segment の代表時刻
- `uri`: Chat 側で source を追跡できる URI

read scope:

- `include_owner_containment: false`: binding された MemorySpace だけを読む。
- `include_owner_containment: true`: owner principal containment をたどる。user owner なら所属 team / organization、team owner なら containing organization の MemorySpace が候補になる。

Memory がしないこと:

- Chat の DB や message schema を読む。
- Chat の channel / DM / thread を canonical Memory object として扱う。
- `app_id = chat` だけを理由に extraction、policy、retrieval を構造分岐する。
- Chat 側 membership filter を信頼して unauthorized evidence を返す。
