# バックエンド設計

## 技術スタック

| 項目 | 選定 | 理由 |
|------|------|------|
| ランタイム | Node.js 22 (LTS) | |
| 言語 | TypeScript | |
| フレームワーク | Hono | 軽量・型安全。ローカル Node.js での起動に最適 |
| DB | SQLite（ローカルファイル） | ローカル専用ツールのためシングルファイルで十分。将来の PostgreSQL 移行を阻害しない設計 |
| ORM | Drizzle ORM | SQLite / PostgreSQL 両対応、型安全 |
| ジョブスケジューラー | node-cron (MVP) → BullMQ (将来) | MVP は cron で十分 |
| YouTube | googleapis | Google 公式 npm パッケージ |
| RSS | rss-parser | |

---

## API エンドポイント一覧

### チャンネル管理

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/channels` | チャンネル一覧取得 |
| POST | `/api/channels` | チャンネル登録（URL から自動判定） |
| PATCH | `/api/channels/:id` | チャンネル設定更新（ポリシー・有効化等） |
| DELETE | `/api/channels/:id` | チャンネル削除 |
| POST | `/api/channels/:id/crawl` | 手動クロール実行 |

### エピソード管理

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/channels/:id/episodes` | エピソード一覧（チャンネル別） |
| PATCH | `/api/episodes/:id` | エピソード設定更新（disabled 等） |

### キュー

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/queue` | 現在のキュー取得（上位N件） |
| GET | `/api/queue/next` | 次のエピソードを取得（視聴履歴に記録後） |
| POST | `/api/queue/rebuild` | キュー手動再構築 |
| PATCH | `/api/queue/reorder` | キュー手動並び替え |

### 視聴履歴

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/history` | 視聴履歴一覧 |
| POST | `/api/history` | 視聴記録の作成・更新 |

### フィルタ

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/filters` | フィルタルール一覧 |
| POST | `/api/filters` | フィルタルール追加 |
| DELETE | `/api/filters/:id` | フィルタルール削除 |
| GET | `/api/mutes` | ミュートリスト取得 |
| POST | `/api/mutes` | ミュート追加 |
| DELETE | `/api/mutes/:id` | ミュート削除 |

### 設定

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/api/settings` | アプリ設定取得 |
| PATCH | `/api/settings` | アプリ設定更新 |

---

## Channel Crawler

### URL 判定ロジック

```
入力 URL
  ├─ youtube.com/channel/UC*    → YouTube チャンネル
  ├─ youtube.com/@handle        → YouTube チャンネル (handle)
  ├─ youtube.com/c/*            → YouTube チャンネル (legacy)
  └─ その他                     → RSS フィードとして試みる
                                   └─ Content-Type が application/rss+xml または application/atom+xml → RSS 確定
                                   └─ HTML ページ → <link rel="alternate" type="application/rss+xml"> を検索
```

### YouTube クロール

1. `youtube.googleapis.com/youtube/v3/channels` でチャンネル情報取得
2. `youtube.googleapis.com/youtube/v3/playlistItems` でアップロード再生リストを取得
3. 取得した動画 ID を `episodes` テーブルへ upsert
4. 動画の duration は `youtube.googleapis.com/youtube/v3/videos` で取得（quotaコスト考慮）

**API Quota 対策**

- `playlistItems.list` は 1 ページ50件まで。`nextPageToken` でページネーション
- 初回クロール: 最大200件（4ページ）まで取得
- 差分クロール: `publishedAfter` フィルタで直近の新着のみ取得

### RSS クロール

1. `rss-parser` で `<item>` を解析
2. `<enclosure url>` が存在する場合は音声/動画 URL を保存
3. `<itunes:duration>` または `<duration>` から再生時間を取得

---

## Queue Builder

### キュー生成アルゴリズム（MVP）

```
1. 全チャンネルのうち有効なものを取得
2. チャンネルごとにエピソードを取得
   - disabled = false
   - 視聴履歴に "completed" がない
3. チャンネルのポリシーに従い並び替え
   - "newest_first": published_at DESC
   - "oldest_first": published_at ASC
4. チャンネルをラウンドロビンで交互にキューへ追加
   （同一チャンネルが連続しないように）
5. 上位 N 件（デフォルト: 50）を返す
```

**将来拡張**: 優先度スコアリング、視聴率・再生時間による重み付け

---

## History Recorder

視聴履歴エントリの `status` フィールド:

| 値 | 意味 |
|----|------|
| `started` | 再生開始済み（進捗不明） |
| `in_progress` | 途中（`progress_seconds` に現在地を保存） |
| `completed` | 見終わり（`onEnded` 検知 or ザッピングで手動スキップ） |
| `skipped` | ザッピングにより自動スキップ |

---

## Episode Filter 判定ロジック

エピソード取得後、以下の順番でフィルタを適用し `is_playable` フラグを更新:

```
1. チャンネルが disabled → false
2. エピソード自体が disabled → false
3. ミュートリスト（チャンネル単位） → false
4. ミュートリスト（エピソード単位） → false
5. 正規表現フィルタ（タイトルに対して全パターンを試行） → いずれかにマッチ → false
6. 視聴履歴に "completed" がある → false
7. 上記すべてをパス → true
```

---

## スケジューラー

| ジョブ | 頻度 | 内容 |
|--------|------|------|
| 差分クロール | 毎時0分 | 全チャンネルの新着エピソードを取得 |
| キュー再構築 | 毎時5分（クロール後） | Queue Builder を実行し DB のキューを更新 |
| フィルタ再評価 | クロール後 | 新着エピソードにフィルタを適用 |

---

## プロキシエンドポイント（UA 変更対応）

```
GET /api/proxy?url=<encoded-url>&ua=<encoded-ua>
```

- サーバーサイドで指定の UA を設定してコンテンツをフェッチ
- レスポンスをそのままクライアントへ転送
- 対象: RSS フィード取得、一部メタデータ取得
- セキュリティ: 許可 URL ホストのホワイトリストを設ける（SSRF 対策）

---

## 環境変数

ローカル専用ツールのため、`.env` ファイルで管理する（Git 管理外）。

| 変数名 | 説明 |
|--------|------|
| `YOUTUBE_API_KEY` | YouTube Data API v3 のキー |
| `DATABASE_PATH` | SQLite ファイルパス（例: `./data/zapping.db`） |
| `PROXY_ALLOWED_HOSTS` | プロキシ許可ホスト（カンマ区切り）。SSRF 対策 |
| `PORT` | API サーバーポート（デフォルト: 3001） |
| `FRONTEND_ORIGIN` | フロントエンドのオリジン（CORS 設定用。例: `http://localhost:3000`） |
