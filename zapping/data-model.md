# データモデル設計

## 使用 DB

SQLite（ローカルファイル）。Drizzle ORM で型安全に操作する。
ファイルパスは環境変数 `DATABASE_PATH` で指定（デフォルト: `./data/zapping.db`）。

---

## テーブル一覧

### `channels` — チャンネル・番組

```sql
CREATE TABLE channels (
  id            TEXT PRIMARY KEY,          -- UUID
  name          TEXT NOT NULL,             -- 表示名
  source_type   TEXT NOT NULL,             -- 'youtube' | 'rss'
  source_url    TEXT NOT NULL UNIQUE,      -- YouTube チャンネル URL or RSS フィード URL
  external_id   TEXT,                      -- YouTube: チャンネル ID (UC...) / RSS: feed URL
  order_policy  TEXT NOT NULL DEFAULT 'newest_first',
                                           -- 'newest_first' | 'oldest_first'
  is_enabled    INTEGER NOT NULL DEFAULT 1, -- 0: 無効
  last_crawled_at TEXT,                    -- ISO8601
  created_at    TEXT NOT NULL,             -- ISO8601
  updated_at    TEXT NOT NULL              -- ISO8601
);
```

### `episodes` — エピソード（動画・音声）

```sql
CREATE TABLE episodes (
  id              TEXT PRIMARY KEY,         -- UUID
  channel_id      TEXT NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  title           TEXT NOT NULL,
  description     TEXT,
  source_url      TEXT NOT NULL,            -- 再生 URL（YouTube: watch URL, Podcast: enclosure URL）
  external_id     TEXT,                     -- YouTube: video ID / RSS: <guid>
  thumbnail_url   TEXT,
  duration_seconds INTEGER,                 -- 再生時間（秒）。取得できない場合 NULL
  published_at    TEXT,                     -- ISO8601
  is_disabled     INTEGER NOT NULL DEFAULT 0, -- 手動除外
  is_playable     INTEGER NOT NULL DEFAULT 1, -- フィルタ適用後の再生可否フラグ
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  UNIQUE (channel_id, external_id)
);
```

### `queue_items` — 再生キュー

```sql
CREATE TABLE queue_items (
  id          TEXT PRIMARY KEY,             -- UUID
  episode_id  TEXT NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
  position    INTEGER NOT NULL,             -- キュー内の順番（小さいほど先）
  added_at    TEXT NOT NULL,               -- ISO8601
  UNIQUE (episode_id)                      -- 同じエピソードはキューに1回まで
);
```

### `watch_history` — 視聴履歴

```sql
CREATE TABLE watch_history (
  id               TEXT PRIMARY KEY,        -- UUID
  episode_id       TEXT NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
  status           TEXT NOT NULL,           -- 'started' | 'in_progress' | 'completed' | 'skipped'
  progress_seconds INTEGER,                 -- 途中停止時の再生位置（秒）
  played_at        TEXT NOT NULL,           -- 再生開始日時 ISO8601
  finished_at      TEXT                     -- 再生終了日時 ISO8601（完了・スキップ時）
);
```

### `filters` — 正規表現フィルタ

```sql
CREATE TABLE filters (
  id          TEXT PRIMARY KEY,             -- UUID
  channel_id  TEXT REFERENCES channels(id) ON DELETE CASCADE,
                                            -- NULL の場合は全チャンネルに適用
  pattern     TEXT NOT NULL,               -- 正規表現文字列
  note        TEXT,                        -- メモ
  is_enabled  INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL
);
```

### `mutes` — ミュートリスト

```sql
CREATE TABLE mutes (
  id          TEXT PRIMARY KEY,             -- UUID
  mute_type   TEXT NOT NULL,               -- 'channel' | 'episode' | 'keyword'
  target_id   TEXT,                        -- channel.id or episode.id（mute_type が channel/episode の場合）
  keyword     TEXT,                        -- mute_type が 'keyword' の場合
  note        TEXT,
  created_at  TEXT NOT NULL
);
```

### `settings` — アプリ設定（KV ストア）

```sql
CREATE TABLE settings (
  key         TEXT PRIMARY KEY,
  value       TEXT NOT NULL,               -- JSON 文字列
  updated_at  TEXT NOT NULL
);
```

主要なキー:

| キー | 型 | デフォルト | 説明 |
|-----|-----|-----------|------|
| `zapping_interval_seconds` | number | `1800` | ザッピング間隔（秒）。0で無効 |
| `zapping_grace_seconds` | number | `10` | 「続きを見る？」表示時間（秒） |
| `default_playback_rate` | number | `1.0` | デフォルト再生速度 |
| `proxy_user_agent` | string | `""` | プロキシフェッチ時の UA（空文字はデフォルト UA） |
| `queue_size` | number | `50` | キューの最大件数 |

---

## インデックス

```sql
-- エピソード検索の高速化
CREATE INDEX idx_episodes_channel_id ON episodes(channel_id);
CREATE INDEX idx_episodes_published_at ON episodes(published_at);
CREATE INDEX idx_episodes_is_playable ON episodes(is_playable);

-- 視聴履歴の高速検索
CREATE INDEX idx_watch_history_episode_id ON watch_history(episode_id);
CREATE INDEX idx_watch_history_status ON watch_history(status);

-- キューの順序取得
CREATE INDEX idx_queue_items_position ON queue_items(position);
```

---

## ER 図（概略）

```
channels ──< episodes ──< queue_items
               │
               └──< watch_history

channels ──< filters
mutes（独立。channel_id / episode_id で episodes / channels を参照）
settings（独立 KV）
```

---

## マイグレーション方針

- Drizzle ORM の `drizzle-kit` でマイグレーションファイルを管理
- アプリ起動時に自動マイグレーション（`migrate()` を起動スクリプトに組み込む）
- SQLite ファイルはホスト側ディレクトリにマウント（Docker 利用時）し、コンテナ再起動でもデータが消えない
