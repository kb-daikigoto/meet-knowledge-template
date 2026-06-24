---
name: meet-knowledge-schedule
description: meet-knowledge をいつ実行するか（曜日・時刻）を設定する。クラウド routine（PC オフでも動く）を schedule スキルで作成・更新する。デフォルトは平日 16:00 JST。「meet-knowledge のスケジュール設定」「いつ実行するか変えたい」「/meet-knowledge のスケジュール」で起動。
---

# meet-knowledge スケジュール設定

`meet-knowledge` を定期実行するクラウド routine を作る／更新する。
PC オフでも動く（クラウド実行・Google Drive コネクタは既定で継承）。

## デフォルト
- **平日（月〜金）16:00 JST**（cron: `0 16 * * 1-5`, timezone `Asia/Tokyo`）

## 前提
- `/meet-knowledge:setup` 実行済み（`~/.config/meet-knowledge/config.json` がある）

## ワークフロー

### 1. 実行タイミングを確認
ユーザーに希望の曜日・時刻を聞く（`AskUserQuestion` 可）。未指定ならデフォルト（平日 16:00 JST）。
よくある選択肢:
- 平日 16:00（デフォルト）
- 毎日 朝 9:00
- 平日 朝夕（例: 9:00 と 18:00 の2本）
- 1時間ごと（業務時間帯のみ等）

希望を **cron 式（Asia/Tokyo）** に変換する。例:
- 平日 16:00 → `0 16 * * 1-5`
- 毎日 9:00 → `0 9 * * *`
- 平日 9:00 と 18:00 → 2本（`0 9 * * 1-5`, `0 18 * * 1-5`）

### 2. Chatwork 認証情報を読む
クラウド routine は config.json を読めないので、環境変数として渡す:
```sh
CFG="$HOME/.config/meet-knowledge/config.json"
python3 -c "import json;c=json.load(open('$CFG'));print(c['chatwork_token']);print(c['my_chat_room_id'])"
```
（取得した2値を routine 環境変数 `CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID` に設定する）

### 3. routine を作成（schedule スキルへ委譲）
`schedule` スキルを使って、以下の routine を作る/更新する:
- 名前: `meet-knowledge`
- 実行コマンド: `/meet-knowledge`
- スケジュール: 手順1の cron（timezone Asia/Tokyo）
- 環境変数: `CHATWORK_TOKEN`, `MY_CHAT_ROOM_ID`
- コネクタ: **Google Drive を含める**（既定で含まれる。明示的に外さないこと）
- ネットワーク: **api.chatwork.com への egress を許可**（Chatwork へは直接 HTTPS で通信するため）

### 4. 確認
作成した routine の名前・スケジュール・次回実行時刻を報告。
変更や停止は再度このスキル、または `schedule` で行えることを伝える。

## 注意
- スケジュールは Asia/Tokyo 基準で設定する
- routine 環境変数は厳密な秘密ストアではない旨を一言添える（Chatwork トークンが入るため）。
  気になる場合はローカル `/loop 1h /meet-knowledge` 運用（PC 起動中）も選べる
- 配布時は各自が setup → schedule を一度ずつ行えばよい
