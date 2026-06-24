# meet-knowledge（テンプレート）

Google Meet の会議メモ（要約＋文字起こし）から「コードに残らない意思決定」を自動検出し、
**Chatwork のマイチャットで承認したものだけ**を知識ベースに保存する、Claude Code 向けのテンプレートです。

> このリポジトリは **GitHub テンプレート**です。右上の **「Use this template」** から自分のリポジトリを作って使ってください。
> 秘密情報（API トークン等）はリポジトリに含まれていません。各自のセットアップ時にローカルへ保存されます。

## 何ができるか

```
Google Meet（要約 ON）
  └ Drive / Meet Recordings に「Gemini によるメモ」Doc（要約＋文字起こしを内包）
        │
   [検出・投稿] 例: 平日 16:00 … 未処理会議を検出 → 候補をマイチャットに番号付き投稿
        │
   あなたが番号で返信（例: 1,3 / 全部 / なし）
        │
   [保存] 例: 毎日 0:00 … 返信を読み、承認分だけ保存 → 完了をマイチャットに通知
        │
   知識ベース（出典 Doc URL・検出根拠つき Markdown が蓄積）
```

設計の要点:

- **承認は Chatwork で番号を返すだけ**（GitHub PR を使わない）
- **PC オフでも動く** — クラウド routine が、認可済みの Google Drive コネクタを継承して Drive を読む（サービスアカウント不要）
- **状態管理は Chatwork のメッセージマーカー**（`[mk:<docId>]` / `[mk-done:<docId>]`）。専用 DB を持たない
- **生の文字起こしは Drive にのみ存在** — 検出処理の一時領域を除き、新しい永続ストレージには保存しない。保存するのは検出済みナレッジのみ

## 構成

```
.claude/skills/
  meet-knowledge/SKILL.md          本体：検出 → Chatwork 投稿 → 番号承認 → 保存
  setup/SKILL.md                   /setup：Meet フォルダ URL と Chatwork トークンの登録
  meet-knowledge-schedule/SKILL.md クラウド routine の作成・実行タイミング設定
knowledge/                         保存されたナレッジの出力先
examples/
  config.example.json              設定ファイルの形（秘密情報は入れない）
  sample-knowledge.md              保存されるナレッジの例（架空）
```

## 前提

- Claude Code
- claude.ai で **Google Drive コネクタ**を連携済み（Meet の要約/文字起こしを読むため）
- **Chatwork アカウント**と API トークン
- Google Meet 側で**文字起こし／要約（メモ）を ON**にしておく（メモ Doc が Drive の「Meet Recordings」に保存される）

## 使い方（各自 1 回）

1. **「Use this template」** でこのリポジトリを複製（private 推奨）
2. 複製した repo を clone し、Claude Code で開く
3. **`/setup`** — Meet Recordings フォルダの URL と Chatwork API トークンを登録（room_id・folder_id を取得し `~/.config/meet-knowledge/config.json` に 600 で保存。すでに設定済みでも上書き実行）
4. `/web-setup` — クラウド routine が repo を読めるよう GitHub を接続（PC オフ運用に必要）
5. `/meet-knowledge-schedule` — クラウド routine（検出・投稿／保存）を作成

以降は会議のたびにマイチャットへ候補が届くので、保存したい番号を返信するだけ。
- まず手元で試す: `/meet-knowledge`（必要なら `/loop 1h /meet-knowledge`）

## クラウド routine（PC オフ運用）

`/meet-knowledge-schedule` で次の 2 本を作成します（時刻は変更可）:

| routine | 役割 | 例のスケジュール |
|---|---|---|
| 検出・投稿 | 未処理会議を検出し候補をマイチャットへ投稿（保存しない） | 平日 16:00 |
| 保存 | マイチャットの番号返信を読み、承認分を保存 | 毎日 0:00 |

- Google Drive コネクタは既定で継承（サービスアカウント不要）
- `schedule` の作成 API に環境変数欄が無いため、`CHATWORK_TOKEN` / `MY_CHAT_ROOM_ID` / `MEET_FOLDER_ID` は routine のプロンプトに埋め込む
- `api.chatwork.com` への直接 HTTPS 通信が必要（サンドボックスの egress 可否は初回実行で確認）

## 検出スコープ

検出対象は、本人が **主催した会議**（自分の Meet Recordings フォルダ）。
他人が主催する会議まで拾う展開は、Google Workspace のドメイン委任が別途必要です。

## セキュリティ / 注意

- API トークンは `~/.config/meet-knowledge/config.json`（権限 600）に保存し、**リポジトリにはコミットしない**
- 会議内容には機微情報が含まれ得ます。**生の文字起こしを新しい場所に保存しない**方針を維持してください
- 機密・個人情報は検出候補に含めない方針です
- 業務利用時は、所属組織のクラウド/AI 利用ガイドラインに従ってください

## ライセンス

MIT License（`LICENSE` を参照）
