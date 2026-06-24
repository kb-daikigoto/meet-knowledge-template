---
name: meet-knowledge-setup
description: meet-knowledge の初期セットアップ。Chatwork API トークンを登録し、マイチャットの room_id を取得して設定ファイルに保存する。meet-knowledge を使う前に一度だけ実行する。「meet-knowledge のセットアップ」「Chatwork 連携の設定」「/setup」で起動。
---

# meet-knowledge セットアップ

`meet-knowledge` がナレッジ候補を **Chatwork のマイチャット**に送るために、
Chatwork API トークンを登録する。**一度だけ**実行すればよい。

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
ユーザーに次を伝える:

> Chatwork の API トークンは、Chatwork 画面右上の「サービス連携」→「API Token」から取得できます。
> （取得 URL: https://www.chatwork.com/service/packages/chatwork/subpackages/api/token.php ）
> 取得したトークンをこのチャットに貼り付けてください。トークンは `~/.config/meet-knowledge/config.json`（本人のみ読める権限）に保存し、画面には表示し直しません。

### 2. トークンを受け取る
- ユーザーが貼り付けたトークンを受け取る
- **トークンを会話やログにそのまま再表示しない**。以降はファイルに保存して扱う

### 3. トークンを検証し、必要な ID を取得する
Bash で Chatwork API を呼ぶ（`<TOKEN>` は受け取った値に置換）:

```sh
# 認証確認 & account_id 取得
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/me
```
- 200 で `account_id` が返れば OK。401/403 ならトークンが無効 → ユーザーに再入力を依頼
- 次にマイチャットの room_id を取得:
```sh
# rooms の中で type == "my" がマイチャット
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/rooms
```
- `type` が `"my"` の room の `room_id` を採用する

### 3.5 Meet Recordings フォルダ ID を取得する
接続済みの Google Drive コネクタで、ユーザー本人が所有する「Meet Recordings」フォルダを検索し、
その `id` を `meet_folder_id` として採用する。

`search_files` クエリ例:
`mimeType = 'application/vnd.google-apps.folder' and title contains 'Meet Recordings' and owner = 'me'`

- 複数ヒットしたら、最新のもの、またはユーザーに確認して 1 つ選ぶ
- 見つからなければユーザーにフォルダ ID を直接聞く（Meet の録画/文字起こし保存先フォルダ）

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
（ヒアドキュメントに実値を埋めて書き込む。トークンを echo で標準出力に出さない）

### 5. 疎通テスト（任意・推奨）
ユーザーに確認のうえ、マイチャットにテスト投稿してよいか聞いてから:
```sh
TOKEN=$(python3 -c "import json;print(json.load(open('$HOME/.config/meet-knowledge/config.json'))['chatwork_token'])")
RID=$(python3 -c "import json;print(json.load(open('$HOME/.config/meet-knowledge/config.json'))['my_chat_room_id'])")
curl -s -X POST -H "X-ChatWorkToken: $TOKEN" \
  -d "body=[info][title]meet-knowledge[/title]セットアップ完了。ここにナレッジ候補が届きます。[/info]" \
  "https://api.chatwork.com/v2/rooms/$RID/messages"
```

### 6. 完了報告
- 保存先パスと、マイチャットの room_id（トークンは伏せる）を報告
- 次のステップを案内: `/meet-knowledge` で実行、`/loop 1h /meet-knowledge` で定期実行、
  または `schedule` でクラウド routine 化（PC オフ）

## クラウド routine（PC オフ）で使う場合の補足
クラウド routine は設定ファイルを読めないため、`schedule` の routine 環境変数に
`CHATWORK_TOKEN`・`MY_CHAT_ROOM_ID`・`MEET_FOLDER_ID` を設定する（routine 編集画面の環境変数）。
`meet-knowledge` skill は「config.json が無ければ環境変数を見る」順で動く。

## 注意
- **トークンは秘密情報**。会話に再表示しない／git にコミットしない／config.json は 600
- トークンは利用者本人の Chatwork アカウントのもの。第三者と共有しない
