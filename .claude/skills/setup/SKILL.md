---
name: setup
description: meet-knowledge の初期セットアップ。Chatwork API トークンを登録し、マイチャット room_id と Meet Recordings フォルダ ID を取得して保存。続けて、このデバイス（このリポジトリ・このアカウント）用に Claude のクラウド routine（検出・投稿／保存）を自動作成する。「/setup」「meet-knowledge のセットアップ」「Chatwork 連携」で起動。
---

# /setup — meet-knowledge セットアップ＆ルーティン作成

`/setup` 一回で次を行う:

1. **Chatwork API トークンを登録**（マイチャット room_id・Meet Recordings フォルダ ID を自動取得して保存）
2. **このデバイス用に Claude のクラウド routine を作成**（検出・投稿／保存の 2 本。PC オフでも動く）

## 設定ファイル
- パス: `~/.config/meet-knowledge/config.json`（パーミッション 600、git にコミットしない）
- 形式:
  ```json
  {
    "chatwork_token": "<API トークン>",
    "account_id": <自分の account_id>,
    "my_chat_room_id": <マイチャットの room_id>,
    "meet_folder_id": "<Meet Recordings フォルダ ID>"
  }
  ```

## ワークフロー

### 1. トークンの取得方法を案内する
ユーザーに伝える:
> Chatwork の API トークンは「サービス連携」→「API Token」から取得できます。
> （https://www.chatwork.com/service/packages/chatwork/subpackages/api/token.php ）
> 取得したトークンを貼り付けてください。トークンは `~/.config/meet-knowledge/config.json`（本人のみ読める権限）に保存し、画面には再表示しません。

### 2. トークンを受け取る
- 受け取ったトークンを会話やログに再表示しない。以降はファイルに保存して扱う

### 3. トークンを検証し ID を取得する
```sh
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/me      # account_id
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/rooms   # type=="my" の room_id
```
- 401/403 ならトークン無効 → 再入力を依頼

### 3.5 Meet Recordings フォルダ ID を取得する
接続済み Google Drive コネクタで本人所有の「Meet Recordings」フォルダを検索し `id` を採用:
`mimeType = 'application/vnd.google-apps.folder' and title contains 'Meet Recordings' and owner = 'me'`
- 複数なら最新／ユーザー確認。見つからなければフォルダ ID を直接聞く

### 4. 設定を保存する
```sh
umask 077
cat > ~/.config/meet-knowledge/config.json <<'JSON'
{
  "chatwork_token": "<TOKEN>",
  "account_id": <ACCOUNT_ID>,
  "my_chat_room_id": <MY_CHAT_ROOM_ID>,
  "meet_folder_id": "<MEET_FOLDER_ID>"
}
JSON
chmod 600 ~/.config/meet-knowledge/config.json
```
（実値を埋めて書き込む。トークンを echo で標準出力に出さない）

### 5. 疎通テスト
ユーザー確認のうえ、マイチャットへテスト投稿:
```sh
TOKEN=$(python3 -c "import json;print(json.load(open('$HOME/.config/meet-knowledge/config.json'))['chatwork_token'])")
RID=$(python3 -c "import json;print(json.load(open('$HOME/.config/meet-knowledge/config.json'))['my_chat_room_id'])")
curl -s -X POST -H "X-ChatWorkToken: $TOKEN" \
  -d "body=[info][title]meet-knowledge[/title]セットアップ完了。ここにナレッジ候補が届きます。[/info]" \
  "https://api.chatwork.com/v2/rooms/$RID/messages"
```

### 6. クラウド routine を作成（このデバイスのセットアップ）
**`meet-knowledge-schedule` スキルの手順**に従い、このデバイス用に 2 本のクラウド routine を作成する。
スケジュールは**既定（検出・投稿 平日16:00 JST ／ 保存 毎日0:00 JST）**。ユーザーが希望すれば変更可。

作成時に必須の値:
- **repo URL**: 現在のリポジトリ。`git remote get-url origin` を https 形式に正規化（`git@github.com:O/R.git` → `https://github.com/O/R`）
- **埋め込む設定**: routine プロンプトに `CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID` / `MEET_FOLDER_ID` と repo URL を埋める（schedule の作成 API に環境変数欄が無いため）
- **コネクタ**: Google Drive を添付
- GitHub 未接続なら、先に **`/web-setup`** を実行するよう案内（未接続だと routine が repo を clone できない）

**冪等性**: `RemoteTrigger {action:"list"}` で既存を確認し、`meet-knowledge` / `meet-knowledge-save` が既にあれば **create でなく update**（二重作成しない）。

作成する 2 本（cron は UTC。JST から 9 時間引く）:
| 名前 | 役割 | cron(UTC) | JST |
|---|---|---|---|
| `meet-knowledge` | 検出 → 候補をマイチャットへ投稿（フェーズB のみ） | `0 7 * * 1-5` | 平日16:00 |
| `meet-knowledge-save` | 番号返信を読み承認分を保存（フェーズA のみ） | `0 15 * * *` | 毎日0:00 |

### 7. 完了報告
- 設定保存先（トークンは伏せる）・マイチャット room_id・Meet フォルダ ID
- 作成/更新した routine 名・スケジュール・次回実行・管理 URL
- 以降は会議のたびに候補が届くので番号で返信、を案内

## 注意
- **トークンは秘密情報**。会話に再表示しない／git にコミットしない／config.json は 600
- トークンは利用者本人の Chatwork アカウントのもの。第三者と共有しない
- 業務利用時は所属組織のクラウド/AI 利用ガイドラインに従う
