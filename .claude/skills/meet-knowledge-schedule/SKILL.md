---
name: meet-knowledge-schedule
description: meet-knowledge をいつ実行するか（曜日・時刻）を設定し、クラウド routine（PC オフでも動く）を作成・更新する。デフォルトは「検出・投稿＝平日16:00 JST」「保存＝毎日0:00 JST」の2本。「meet-knowledge のスケジュール設定」「いつ実行するか変えたい」で起動。
---

# meet-knowledge スケジュール設定（クラウド routine の作成）

`meet-knowledge` を定期実行するクラウド routine を作る／更新する。PC オフでも動く。
承認の都合上、**検出・投稿**と**保存**を別スケジュールの 2 本に分ける。

## デフォルト（JST → cron は UTC で指定。JST から 9 時間引く）
| routine | 役割 | JST | cron(UTC) |
|---|---|---|---|
| `meet-knowledge` | 検出 → 候補をマイチャットへ投稿（フェーズB） | 平日 16:00 | `0 7 * * 1-5` |
| `meet-knowledge-save` | 番号返信を読み承認分を保存（フェーズA） | 毎日 0:00 | `0 15 * * *` |

## 前提
- `/setup` 実行済み（`~/.config/meet-knowledge/config.json` がある）
- `/web-setup` 済み（クラウド routine が repo を clone できる）

## ワークフロー

### 1. 実行タイミングを確認
ユーザーに希望を聞く（未指定はデフォルト）。JST で受けて **UTC に変換**して cron 化する
（例: 平日16:00 JST = `0 7 * * 1-5`）。最小間隔は 1 時間。

### 2. 必要な値を集める
- config から: `chatwork_token` / `my_chat_room_id` / `meet_folder_id` / `knowledge_repo`(任意)
  ```sh
  CFG="$HOME/.config/meet-knowledge/config.json"
  python3 -c "import json;c=json.load(open('$CFG'));print(c.get('chatwork_token',''));print(c.get('my_chat_room_id',''));print(c.get('meet_folder_id',''));print(c.get('knowledge_repo',''))"
  ```
- repo URL: `git remote get-url origin` を https 形式に正規化（`git@github.com:O/R.git` → `https://github.com/O/R`）

### 3. routine を作成／更新（`schedule` スキル＝RemoteTrigger を利用）
**冪等**: `RemoteTrigger {action:"list"}` で既存を確認し、`meet-knowledge` / `meet-knowledge-save`
が既にあれば create でなく **update**。

各 routine に:
- 実行 repo: 手順2の repo URL（sources）
- コネクタ: **Google Drive を含める**（既定で継承。外さない）
- **プロンプトに値を埋め込む**（schedule の create API に環境変数欄が無いため）:
  `CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID` / `MEET_FOLDER_ID`、および保存先 `KNOWLEDGE_REPO`(あれば)
- プロンプトは「`.claude/skills/meet-knowledge/SKILL.md` を読み、該当フェーズのみ実行」と指示
- ネットワーク: `api.chatwork.com` への egress が必要（初回実行で可否確認）

### 保存先のルール（重要）
保存 routine（フェーズA）の保存先は **`knowledge-base:save` または `KNOWLEDGE_REPO` の知識ベース repo のみ**。
**スキル一式が入っている起動元 repo には絶対に書き込まない／push しない。**
どちらも無ければ保存せず、マイチャットに「保存先が未設定」と通知して候補は未解決のまま残す。

### 4. 確認
作成/更新した routine 名・スケジュール・次回実行・管理 URL（`https://claude.ai/code/routines/<id>`）を報告。

## 注意
- スケジュールは JST で受け、cron は UTC に変換する
- routine のプロンプト/環境は厳密な秘密ストアではない（Chatwork トークンが入る）。
  気になる場合はローカル `/loop 1h /meet-knowledge`（PC 起動中）も選べる
- 配布時は各自が `/setup` →（`/web-setup`）→ このスキル、を一度ずつ行う
