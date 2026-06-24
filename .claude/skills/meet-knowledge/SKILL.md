---
name: meet-knowledge
description: Google Meet の文字起こし・要約（Drive の Meet Recordings に保存された「Gemini によるメモ」Doc）から、まだ検出していない会議のナレッジを検出し、候補を Chatwork のマイチャットに送る。ユーザーが番号で返信したものを知識ベースへ保存・push する。定期実行は /loop 1h /meet-knowledge またはクラウド routine（schedule）で行う。事前に /meet-knowledge:setup が必要。
---

# meet-knowledge — Meet 文字起こしからのナレッジ自動検出・Chatwork 承認・保存

Drive の Meet Recordings の「Gemini によるメモ」Doc を読み、未処理のものを検出し、
**候補を Chatwork マイチャットに投稿**。ユーザーが**番号で返信**したものだけ知識ベースへ保存する。

セッション内の対話に依存しないので、PC オフのクラウド routine でも回る（承認は Chatwork 上で非同期に行う）。

## 前提
- `/meet-knowledge:setup` 実行済み（Chatwork トークン設定済み）
- Google Drive コネクタが認可済み（クラウド routine でも既定で継承される）

## 設定

| 項目 | 値 / 取得元 |
|---|---|
| `MEET_FOLDER_ID` | `~/.config/meet-knowledge/config.json` の `meet_folder_id`（無ければ環境変数 `MEET_FOLDER_ID`）。各自の Meet Recordings フォルダ ID で、`/meet-knowledge-setup` が設定する |
| `NOTE_NAME_SUBSTR` | `Gemini によるメモ` |
| Chatwork トークン / room_id | `~/.config/meet-knowledge/config.json`。無ければ環境変数 `CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID`（クラウド routine 用） |
| `MAX_PER_RUN` | 5 |
| 保存先 | `knowledge-base:save` が使えればそれ。使えない場合はこのリポジトリの `knowledge/` に Markdown を commit/push（クラウド routine は基本こちら） |

Chatwork 認証情報の読み込み（Bash）:
```sh
CFG="$HOME/.config/meet-knowledge/config.json"
if [ -f "$CFG" ]; then
  TOKEN=$(python3 -c "import json;print(json.load(open('$CFG'))['chatwork_token'])")
  RID=$(python3 -c "import json;print(json.load(open('$CFG'))['my_chat_room_id'])")
  FOLDER=$(python3 -c "import json;print(json.load(open('$CFG')).get('meet_folder_id',''))")
else
  TOKEN="$CHATWORK_TOKEN"; RID="$MY_CHAT_ROOM_ID"; FOLDER="$MEET_FOLDER_ID"
fi
```

## ワークフロー（毎回これを実行）

### フェーズ A: 返信を回収して保存（前回投稿の承認処理）
1. マイチャットの最近のメッセージを取得:
   `curl -s -H "X-ChatWorkToken: $TOKEN" "https://api.chatwork.com/v2/rooms/$RID/messages?force=1"`
2. **未解決の候補投稿**を探す: 本文に `[mk:<docId> batch:<n>]` マーカーを含み、
   その後に `[mk-done:<docId> batch:<n>]`（保存済みマーカー）が**まだ無い**もの
3. その候補投稿より後に来た、**番号だけ/番号の羅列**（例 `1,3` `1 3` `13` `全部` `なし`）のメッセージを「選択」とみなす
   - `全部`=全件、`なし`/`0`=保存しない
4. 選ばれた候補を保存（承認分のみ）。`knowledge-base:save` が使えればそれを使う。使えない場合はリポジトリの `knowledge/` に Markdown を書いて git add/commit/push（フロントマターに source_doc_url・検出メタ）。内容を充実させるなら docId から Drive の Doc を読んで背景・詳細を補う
5. 保存後、確認を投稿:
   `[mk-done:<docId> batch:<n>] ✅ N件保存しました: <ファイル名...>`（このマーカーで再処理を防ぐ）

### フェーズ B: 新規会議を検出して候補を投稿
1. Drive を検索: `mcp__claude_ai_Google_Drive__search_files` で
   `parentId = 'MEET_FOLDER_ID' and mimeType = 'application/vnd.google-apps.document'`、
   タイトルに `NOTE_NAME_SUBSTR` を含むものだけ
2. **未処理**を判定: マイチャットに `[mk:<docId> ...]` 投稿が既にあれば処理済みとみなしスキップ
   （＝Chatwork を状態ストアとして使う。別途 state ファイルは不要）
3. 未処理を古い順に最大 `MAX_PER_RUN` 件、各 Doc を取得:
   `mcp__claude_ai_Google_Drive__read_file_content`
4. 本文を `文字起こし` 見出しで分割（上=要約、下=文字起こし全文）。タイトル/日時を復元
5. **ナレッジ検出**（下記基準）。候補ごとに 概要 / 判定根拠 / 信頼度 / タグ
6. 候補があればマイチャットに投稿（POST /rooms/$RID/messages, `body=...`）:
   ```
   [info][title]🧠 ナレッジ候補 [mk:<docId> batch:1] <会議名/日時>[/title]
   保存するものを番号で返信してください（例: 1,3 / 全部 / なし）

   1. <候補1タイトル>
      <概要> （根拠: <...> / 信頼度: <...>）
   2. <候補2タイトル>
      ...
   [/info]
   ```
   候補ゼロなら投稿しない（必要なら `[mk:<docId> batch:0] 検出なし` を投稿して再処理回避）

## ナレッジ検出基準（会議向け）
**コードから導出できない、人間の意思決定とその文脈**だけを候補にする。

検出する: 設計/方式の採否理由・設定値やしきい値の根拠・注意点や落とし穴・障害の教訓・
暗黙の前提や制約・「あえて〜しない」不作為の決定・体制や責任分界の理由。

検出しない: 雑談/近況・単なる進捗共有・次回日程や事務連絡・一般的な技術解説・
スペック（API 形式やテーブル定義）・**結論が出ていない持ち帰り**。

注意:
- 要約の「詳細」で当たりをつけ、文字起こし本文で「なぜ/前提」を裏取り
- 「決まったこと」と「保留」を区別し確定した意思決定のみ採用
- **機密・個人情報（人事評価・個人の処遇・未公開数値など）は候補にしない**。迷うものは除外側に倒す

`knowledge-base:save` に渡す内容: タイトル / 背景(どの会議のどの発言か、source Doc URL を含む) / 詳細 / 結論 / タグ。

## 定期実行
- ローカル: `/loop 1h /meet-knowledge`（PC 起動中。毎回フェーズA→Bを実行）
- PC オフ（推奨・2 routine 分割）: `schedule` でクラウド routine を2本作る。
  - **フェーズB専用**（検出・投稿）: 平日 16:00 JST（会議後に候補を投稿）
  - **フェーズA専用**（保存）: 毎日 0:00 JST（夜のうちの番号返信を翌 0:00 に保存）
  - Google Drive コネクタは既定で継承される。Chatwork トークン/room_id は
    **routine のプロンプトに埋め込む**（schedule の create API に環境変数欄が無いため）。
  - api.chatwork.com への直接 HTTPS 通信が必要（サンドボックスの egress 可否は初回実行で確認）。

## 設計メモ
- 生の文字起こしは Drive と一時コンテキストにしか乗らない（新しい永続ストレージに保存しない）
- 状態管理は Chatwork のメッセージ（`[mk:...]` / `[mk-done:...]` マーカー）で行う
- 承認は Chatwork マイチャットでの番号返信（PR は使わない）
