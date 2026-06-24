---
name: setup
description: meet-knowledge のフルセットアップ。Meet Recordings フォルダの URL と Chatwork API トークンを登録し、続けて GitHub 接続（/web-setup）の確認とクラウド routine の作成（/meet-knowledge-schedule 相当）まで行う。すでに設定済みでも毎回実行し上書きする。「/setup」「meet-knowledge のセットアップ」で起動。
---

# /setup — meet-knowledge フルセットアップ

`/setup` 一回で、次を順に行う:

1. **Meet フォルダ URL と Chatwork API トークンを登録**（ユーザーに促すのはこの 2 つだけ）
2. **GitHub 接続を確認**（未接続なら `/web-setup` を案内）
3. **クラウド routine を作成**（検出・投稿／保存の 2 本）

**この処理は、すでに設定済みでも毎回実行して上書きする**（スキップしない）。

## 保存先（config）
- `~/.config/meet-knowledge/config.json`（600、git にコミットしない）
  ```json
  {
    "chatwork_token": "<API トークン>",
    "account_id": <account_id>,
    "my_chat_room_id": <マイチャット room_id>,
    "meet_folder_id": "<Meet Recordings フォルダ ID>",
    "knowledge_repo": "<任意: 知識ベース専用 repo の URL>"
  }
  ```

## ワークフロー

### 1. 2 つの値の入力を促す
1. **Meet Recordings フォルダの URL**（例 `https://drive.google.com/drive/folders/xxxx`）
2. **Chatwork API トークン**（「サービス連携」→「API Token」。
   https://www.chatwork.com/service/packages/chatwork/subpackages/api/token.php ）
   貼り付けてもらう。**トークンは画面に再表示しない**。

### 2. フォルダ URL → folder_id
`/folders/<ID>` の `<ID>` を `meet_folder_id` に（末尾 `?...` は除去）。ID 直貼りならそのまま。

### 3. トークン検証 & ID 取得
```sh
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/me      # account_id
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/rooms   # type=="my" の room_id
```
401/403 なら再入力を依頼。

### 4. 設定を保存（常に上書き）
```sh
umask 077; mkdir -p ~/.config/meet-knowledge
cat > ~/.config/meet-knowledge/config.json <<'JSON'
{ "chatwork_token":"<TOKEN>","account_id":<ACCOUNT_ID>,"my_chat_room_id":<RID>,"meet_folder_id":"<FOLDER_ID>" }
JSON
chmod 600 ~/.config/meet-knowledge/config.json
```
（実値を埋める。トークンを echo しない。`knowledge_repo` が既にあれば残す）

### 5. GitHub 接続を確認（/web-setup）
クラウド routine が repo を clone するには GitHub 接続が必要。
- 接続状況を確認（例: `gh auth status`、または後続の routine 作成時の警告）。
- **未接続なら、ユーザーに `/web-setup` の実行を依頼する**:
  > クラウド routine が GitHub の repo を読めるよう、いったん `/web-setup` を実行して GitHub を連携してください（ブラウザが開きます）。完了したら「done」と教えてください。
- **注意: `/web-setup` はブラウザ認証のため Claude が代行できない。** ユーザーの完了を待ってから次へ進む。

### 6. クラウド routine を作成
**`meet-knowledge-schedule` スキルの手順**に従い、デフォルト 2 本を作成／更新する（冪等: 既存は update）:
- `meet-knowledge`（検出・投稿）= 平日16:00 JST（cron `0 7 * * 1-5` UTC）
- `meet-knowledge-save`（保存）= 毎日0:00 JST（cron `0 15 * * *` UTC）

作成時:
- repo URL は `git remote get-url origin` を https 化して使う
- プロンプトに `CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID` / `MEET_FOLDER_ID`（と `knowledge_repo` があれば `KNOWLEDGE_REPO`）を埋め込む
- Google Drive コネクタを添付
- **保存先は知識ベースのみ。起動元 repo には絶対に push しない**（保存先未設定なら保存せず Chatwork に通知）

### 7. 完了報告
- 設定保存先（トークンは伏せる）・room_id・folder_id
- 作成/更新した routine 名・スケジュール・次回実行・管理 URL
- 以降は会議のたびに候補が届くので番号で返信、を案内

## 注意
- **トークンは秘密情報**。再表示しない／コミットしない／config.json は 600
- `/web-setup` のブラウザ認証だけはユーザー操作が必要（Claude は代行不可）
- すでに設定済みでも毎回実行・上書きする
