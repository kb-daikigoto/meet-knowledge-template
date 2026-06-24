---
name: setup
description: meet-knowledge の初期設定。Meet Recordings フォルダの URL と Chatwork API トークンを登録するだけ。すでに設定済みでも毎回実行し、上書きする。「/setup」「meet-knowledge のセットアップ」「Chatwork 連携」で起動。
---

# /setup — meet-knowledge 初期設定

Meet の文字起こし/要約が保存されるフォルダと、Chatwork API トークンを登録する。
**この処理は、すでに設定済みであっても毎回実行して上書きする**（「設定済みだから」とスキップしない）。

> ルーティン（定期実行）の作成はここでは行わない。`/meet-knowledge-schedule` で設定する。

## 保存先
- `~/.config/meet-knowledge/config.json`（パーミッション 600、git にコミットしない）
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

### 1. 2 つの値の入力を促す
ユーザーに次の 2 つを尋ねる:

1. **Meet Recordings フォルダの URL**
   Google Drive で Meet の録画/文字起こし（「Gemini によるメモ」）が保存されるフォルダを開いた URL。
   例: `https://drive.google.com/drive/folders/xxxxxxxxxxxxxxxxxxxxx`
2. **Chatwork API トークン**
   Chatwork「サービス連携」→「API Token」から取得。
   （https://www.chatwork.com/service/packages/chatwork/subpackages/api/token.php ）
   貼り付けてもらう。**トークンは画面に再表示しない**。

### 2. フォルダ URL から folder_id を取り出す
URL の `/folders/<ID>` の `<ID>` を `meet_folder_id` とする（末尾の `?...` クエリは除去）。
URL でなく ID をそのまま貼られた場合はそのまま使う。

### 3. トークンを検証し ID を取得する
```sh
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/me      # account_id
curl -s -H "X-ChatWorkToken: <TOKEN>" https://api.chatwork.com/v2/rooms   # type=="my" の room_id
```
- 401/403 ならトークン無効 → 再入力を依頼

### 4. 設定を保存する（常に上書き）
既存の config.json があっても上書きする。
```sh
umask 077; mkdir -p ~/.config/meet-knowledge
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

### 5. 完了報告
- 保存先パス・マイチャット room_id・Meet フォルダ ID を報告（トークンは伏せる）
- 次のステップを案内:
  - `/meet-knowledge` … 手動で 1 回実行
  - `/meet-knowledge-schedule` … 定期実行（クラウド routine）を設定（PC オフ運用）

## 注意
- **トークンは秘密情報**。会話に再表示しない／git にコミットしない／config.json は 600
- トークンは利用者本人の Chatwork アカウントのもの。第三者と共有しない
- すでに設定済みでも毎回実行し、上書きする
