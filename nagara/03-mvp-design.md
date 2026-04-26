# 03. MVP 設計

「**まずミニマムに作ってみてから考える**」という依頼方針に従い、
本書は **MVP として実装すべき最小構成** だけを書く。
将来拡張のための抽象化は `data-model` と `PlayerAdapter` インターフェイスの 2 点だけに留め、
他は素直で短いコードで書ける構造にする。

## 1. 全体像

```
┌──────────── Linux PC（自宅 LAN 内、常駐） ────────────┐
│                                                     │
│  ┌──────────────┐    ┌────────────────────────┐     │
│  │ nagara API    │   │ better-sqlite3 (file)  │     │
│  │ (Hono / Node) │←─→│ ./data/nagara.db       │     │
│  └──────┬───────┘    └────────────────────────┘     │
│         │ Static    fetch (RSS, JSON)               │
│         │ files     ↓                               │
│  ┌──────▼───────┐                                  │
│  │ nagara Web   │ ── ▲ public RSS endpoints の取得  │
│  │ (静的 SPA)   │     │                            │
│  └──────────────┘     ▼                            │
│                                                     │
└────────────────────────┬────────────────────────────┘
                         │ HTTP (LAN)
                  ┌──────▼───────┐
                  │ Windows PC   │ ブラウザで http://nagara.local:5173
                  │ (Chrome 等)  │
                  └──────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │ 各サービス（YouTube 等） │ ← ブラウザから直接 iframe / audio
              └──────────────────────────┘
```

ポイント:

- **動画／音声ストリーム自体はサーバーを経由しない。** ブラウザから YouTube／Podcast 配信元へ直接。
  これによりサーバー側は単なる「メタデータ／キュー管理サービス」で済む。
- バックエンドが触る外部通信は、各チャンネルの **新着取得（XML/RSS）のみ**。

## 2. デプロイ戦略

要件「起動が面倒なのは避けたい」を最優先する。

### 採用: Linux PC で systemd 常駐

- `node dist/server.js` を `nagara.service` として登録。
- ブラウザは `http://<linux-ip>:<port>` を開きっぱなしにしておけば、再起動なしで使える。
- フロントとバックエンドは **同一プロセス／同一ポート** で配信する（Hono が静的ファイルもサーブ）。
  → `FRONTEND_ORIGIN`／CORS の設定が要らなくなり、ミニマム化に寄与。

### 不採用: Windows PC で常駐

- 起動毎にコンソールが立つ／タスクトレイ常駐の作り込みが要る。
- 「起動が面倒」を悪化させる。
- それでも単体で動かしたい場合は、同じ Node スクリプトを Windows でも `node dist/server.js` で
  立てれば動く。クロスプラットフォーム差は出ない。

### 不採用: Docker Compose

- ミニマムには重い。SQLite ファイルだけ持てばいいツールに Docker Engine の前提を要求するのは
  「起動が面倒を避けたい」要件に逆行する。
- 代わりに **配布物は単一 Node プロジェクト**（フロント／バックエンドを 1 つの `package.json` に）。

## 3. 採用技術スタック

「ミニマム」と「ローカル単体起動」を最優先。

| レイヤ | 採用 | 不採用と理由 |
|---|---|---|
| バックエンドランタイム | Node.js 22 LTS + TypeScript | — |
| バックエンドフレームワーク | **Hono**（Node アダプタ） | Express でも可だが、静的配信＋ルーティング＋型推論が Hono の方が短く書ける |
| DB | **`better-sqlite3`**（同期ドライバ） | Drizzle / Prisma は MVP には過剰。スキーマも単純、テーブル数が一桁。素の SQL のほうが小さい |
| マイグレーション | 起動時に `CREATE TABLE IF NOT EXISTS` を流すだけ | drizzle-kit / prisma migrate は MVP では不要 |
| RSS パース | `fast-xml-parser` | `rss-parser` よりわずかに軽量、依存も少ない。動作はどちらでも可 |
| スケジューラ | `setInterval`（プロセス内） | node-cron / BullMQ は MVP には大袈裟。1 プロセスに閉じる |
| フロントビルド | **Vite + React + TypeScript** | Next.js は SSR を使わないため過剰。Vite で静的ファイル吐いて Hono が配信 |
| 状態管理 | `useState` / `useReducer` のみ | Zustand / Redux は MVP では不要。画面が 3〜4 枚しかない |
| HTTP クライアント | 素の `fetch` | SWR / React Query は MVP では不要。再取得も画面更新時で足りる |
| スタイル | CSS Modules（ベタ書き） | Tailwind / shadcn は導入コストの方が大きい |

> ここでの方針は **「足りなくなったら入れる」** であって、「最初から将来を見越して全部入れる」ではない。
> Issue 本文の依頼に揃えてある。

## 4. データモデル

SQLite。テーブル 5 個。すべて `id INTEGER PRIMARY KEY AUTOINCREMENT`。
拡張のためのフィールドは確保しつつ、UI を作るのは MVP スコープ内のもののみ。

```sql
-- チャンネル（YouTube チャンネル / Podcast 番組）
CREATE TABLE IF NOT EXISTS channels (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  source_type     TEXT NOT NULL CHECK (source_type IN ('youtube', 'podcast')),
  source_url      TEXT NOT NULL UNIQUE,         -- 元 URL（人間が貼ったもの）
  feed_url        TEXT NOT NULL,                -- 実際にクロールする RSS URL
  external_id     TEXT,                         -- YouTube: UC...
  title           TEXT NOT NULL,
  order_policy    TEXT NOT NULL DEFAULT 'newest_first'
                  CHECK (order_policy IN ('newest_first', 'oldest_first')),
  is_enabled      INTEGER NOT NULL DEFAULT 1,
  last_crawled_at TEXT,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- エピソード（動画 / ポッドキャストの 1 本）
CREATE TABLE IF NOT EXISTS episodes (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  channel_id       INTEGER NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  external_id      TEXT NOT NULL,               -- YouTube: videoId / Podcast: <guid>
  title            TEXT NOT NULL,
  description      TEXT,
  source_type      TEXT NOT NULL,               -- 'youtube' | 'podcast'
  media_url        TEXT,                        -- Podcast: <enclosure url>。YouTube は NULL（videoId で再生）
  duration_seconds INTEGER,
  published_at     TEXT,                        -- ISO8601
  is_disabled      INTEGER NOT NULL DEFAULT 0,  -- 手動 Disabled
  created_at       TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE (channel_id, external_id)
);
CREATE INDEX IF NOT EXISTS idx_episodes_channel    ON episodes(channel_id);
CREATE INDEX IF NOT EXISTS idx_episodes_published  ON episodes(published_at);

-- 視聴履歴（同じエピソードを複数回再生し得るので複数行可）
CREATE TABLE IF NOT EXISTS watch_history (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  episode_id       INTEGER NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
  status           TEXT NOT NULL CHECK (status IN ('started', 'completed', 'skipped')),
  progress_seconds INTEGER,                     -- skipped 時の最終再生位置
  played_at        TEXT NOT NULL DEFAULT (datetime('now')),
  finished_at      TEXT
);
CREATE INDEX IF NOT EXISTS idx_history_episode ON watch_history(episode_id);

-- フィルタ（正規表現／ミュート／キーワードを 1 テーブルにまとめる）
CREATE TABLE IF NOT EXISTS filters (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  filter_type TEXT NOT NULL CHECK (filter_type IN ('regex', 'channel_mute', 'keyword')),
  channel_id  INTEGER REFERENCES channels(id) ON DELETE CASCADE,  -- channel_mute / regex の対象
  pattern     TEXT,                              -- regex / keyword の文字列
  note        TEXT,
  is_enabled  INTEGER NOT NULL DEFAULT 1,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- KV 形式の設定（ザッピング間隔等）
CREATE TABLE IF NOT EXISTS settings (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

設計上のメモ:

- **キューを永続化テーブルにしない。** Issue で「キュー」と呼んでいる対象は
  「いま再生可能なエピソード一覧」を都度計算したものに過ぎない。
  履歴と Disabled とフィルタを引いた SELECT 一発で出せるので、テーブルを増やす必要がない。
  → 実装行数が減り、整合性バグも減る。
- `filters` テーブルでミュート・正規表現・キーワードを 1 つにまとめてあるのは、
  UI を別画面にするのが MVP では過剰だから。型のバリデーションは TS 側で。
- 「投稿者単位ミュート」と「チャンネル単位ミュート」が等価になっているが、
  本ツールの意味では「投稿者 = チャンネル」なので分ける必要が無い。

## 5. バックエンド API（最小セット）

すべて `application/json`。エラーは HTTP ステータス＋ `{ "error": "..." }` を返す。
認証は無し（LAN 限定）。

### 5-1. チャンネル

| Method | Path | 振る舞い |
|---|---|---|
| GET | `/api/channels` | 一覧（`is_enabled`, `last_crawled_at` 含む） |
| POST | `/api/channels` | `{ url }` を受け取り、YouTube / Podcast を自動判定して登録 |
| PATCH | `/api/channels/:id` | `is_enabled` / `order_policy` の更新 |
| DELETE | `/api/channels/:id` | 削除（episodes は ON DELETE CASCADE） |
| POST | `/api/channels/:id/refresh` | 即時クロール（手動） |

URL 自動判定ルール（[02-research.md](./02-research.md) を踏まえる）:

```
入力 url
  ├ youtube.com/channel/UC... または youtube.com/feeds/videos.xml?channel_id=UC...
  │     → channel_id を抽出して feed_url を組み立てる
  ├ youtube.com/@handle 形式
  │     → 当該 URL を取得して "channelId":"UC..." を正規表現で抽出
  └ それ以外
        → そのまま RSS と仮定して fast-xml-parser で読む
        → <itunes:author> 等の存在で podcast 判定
```

### 5-2. キュー（= 「次に流せるエピソード」一覧）

| Method | Path | 振る舞い |
|---|---|---|
| GET | `/api/queue?limit=20` | 次に流せる順に最大 N 件を返す |
| POST | `/api/queue/start` | 1 件を「再生開始」として履歴に `started` を記録し、再生情報を返す |
| POST | `/api/queue/complete` | 「再生完了 / スキップ」を履歴に記録（body に `episodeId` と `status`、必要なら `progressSeconds`） |

キュー生成 SQL（イメージ）:

```sql
SELECT e.*, c.title AS channel_title, c.source_type, c.order_policy
FROM episodes e
JOIN channels c ON c.id = e.channel_id
WHERE c.is_enabled = 1
  AND e.is_disabled = 0
  AND NOT EXISTS (
    SELECT 1 FROM watch_history h
    WHERE h.episode_id = e.id AND h.status = 'completed'
  )
  AND NOT EXISTS (
    SELECT 1 FROM filters f
    WHERE f.is_enabled = 1
      AND (
           (f.filter_type = 'channel_mute' AND f.channel_id = c.id)
        OR (f.filter_type = 'regex'        AND e.title REGEXP f.pattern)
        OR (f.filter_type = 'keyword'      AND e.title LIKE '%' || f.pattern || '%')
      )
  )
ORDER BY ...;
```

並び替えはアプリ側 JS で

1. チャンネルごとにグループ化
2. `order_policy` に従って並び替え
3. チャンネル間でラウンドロビン
4. `limit` で切る

の手順を踏む。SQL に押し込まないのは、可読性とテスト容易性のため。

実装上の注意:

- SQLite には標準で `REGEXP` 演算子が入っていない（演算子だけ予約され、実装は別途必要）。
  `better-sqlite3` を使う前提では、起動時に `db.function('regexp', { deterministic: true }, (pattern, value) => new RegExp(pattern).test(value ?? '') ? 1 : 0)` を登録して有効化する。
- フィルタを SQL の `EXISTS` に押し込むのが嫌なら、`regex` フィルタだけアプリ側で適用する分岐に
  しても良い（その場合は `channel_mute` / `keyword` だけ SQL で先に絞り、結果を JS で正規表現フィルタに掛ける）。MVP ではどちらでも可。

### 5-3. エピソード操作

| Method | Path | 振る舞い |
|---|---|---|
| GET | `/api/channels/:id/episodes` | チャンネル別エピソード一覧（最近 100 件） |
| PATCH | `/api/episodes/:id` | `{ is_disabled }` の更新 |

### 5-4. 履歴

| Method | Path | 振る舞い |
|---|---|---|
| GET | `/api/history?limit=50` | 直近の履歴 |

### 5-5. 設定 / フィルタ

| Method | Path | 振る舞い |
|---|---|---|
| GET / PATCH | `/api/settings` | KV を一括取得・部分更新 |
| GET / POST / DELETE | `/api/filters` | 一覧／追加／削除 |

### 5-6. クロール

| トリガ | 内容 |
|---|---|
| プロセス起動時 | `is_enabled = 1` の全チャンネルを順に 1 回クロール |
| 60 分ごと（`setInterval`） | 同上 |
| `POST /api/channels/:id/refresh` | 当該 1 件のみクロール |

クロール処理:

1. `feed_url` に対し `fetch`（必要に応じて `User-Agent: ${OUTBOUND_USER_AGENT ?? "nagara/0.1"}`）。
2. fast-xml-parser で `<entry>`（YouTube）または `<item>`（Podcast）を取り出す。
3. `(channel_id, external_id)` で UPSERT。
4. 失敗時はログに残してスキップ（1 チャンネルの失敗で全体を止めない）。

## 6. フロントエンド設計

画面は最小 4 つ。すべて 1 ページの SPA としても、別ルートでもよい。

```
/                ← プレイヤー（メイン画面）
/channels        ← チャンネル登録・一覧
/channels/:id    ← エピソード一覧・Disabled トグル
/history         ← 視聴履歴（読み取りのみ）
```

設定（ザッピング間隔・フィルタ）は当面プレイヤー画面右上のドロワーに同居。
画面を増やすほど MVP の重量が増えるため、明示的に 1 ヶ所に集約する。

### 6-1. メインプレイヤー画面の構成

```
┌─────────────────────────────────────────────┐
│ nagara                       [Channels][Hist] │
├─────────────────────────────────────────────┤
│                                              │
│   ┌──────────────────────────────────────┐  │
│   │ Player 領域                          │  │
│   │   - source_type が youtube  → YT iframe│  │
│   │   - source_type が podcast  → <audio>  │  │
│   └──────────────────────────────────────┘  │
│                                              │
│   タイトル / チャンネル名 / 公開日           │
│                                              │
│   [▶/⏸] [⏭次へ] [速度 ×1.0 ▾] [全画面]       │
│                                              │
├─────────────────────────────────────────────┤
│ Up next（キュー先頭 5 件）                   │
└─────────────────────────────────────────────┘
```

### 6-2. PlayerAdapter インターフェイス

「ドメインごとにロード／再生／一時停止／見終わり／速度／フルスクリーンを統一」する要件に対する
唯一の抽象化レイヤー。実装は MVP では 2 つだけ。

```ts
type PlayerEvent =
  | { type: "ended" }
  | { type: "error"; code: string; message: string };

interface PlayerAdapter {
  mount(container: HTMLElement): Promise<void>;
  load(episode: Episode): Promise<void>;
  play(): Promise<void>;
  pause(): Promise<void>;
  setRate(rate: number): Promise<void>;
  getCurrentTime(): number;
  getDuration(): number;
  on(handler: (e: PlayerEvent) => void): void;
  destroy(): void;
}
```

| 実装 | 中身 |
|---|---|
| `YouTubeAdapter` | `<div>` に対し YouTube IFrame Player API で `YT.Player` を生成。`onStateChange` で `ENDED` を `ended` に変換。`onError` を `error` に変換。フルスクリーンは外側コンテナに `requestFullscreen()` |
| `PodcastAudioAdapter` | `<audio>` を `container` 配下に作成。`ended` イベントをそのまま `ended` に変換。`error` イベントを `error` に。`playbackRate` で速度。フルスクリーンは概念的に意味が薄いので、コンテナ拡大表示で代用 |

### 6-3. 自動再生の制御

- 初回起動時は **「Start listening」ボタン** を中央に表示し、ユーザーが押した瞬間に
  最初の `play()` を発火する（[02-research.md](./02-research.md) §1-2 / §5）。
- 連鎖再生（ended → 次のエピソードへ）は同一ジェスチャ文脈内で動くので追加対応不要。
- 連鎖中に Autoplay ブロックされたら、画面中央に「Resume」ボタンを再表示する（フォールバック）。

### 6-4. ザッピング機能

- 設定 `zapping_interval_seconds`（デフォルト 1800 = 30 分）。0 なら無効。
- 各エピソード再生開始時に `setTimeout` を仕込む。
- タイマー満了時、画面下部にオーバーレイ:

  ```
  ┌────────────────────────────┐
  │ そろそろ次に行きます       │
  │ 8 秒後に切り替え            │
  │ [このまま続ける] [いま切替] │
  └────────────────────────────┘
  ```

- 猶予秒数 `zapping_grace_seconds`（デフォルト 10）が経つか「いま切替」を押すと、
  現在エピソードを `skipped`（progress_seconds 付き）として履歴に記録し、次へ。
- 「このまま続ける」を押すとタイマーをリセットして再開。
- `ended` イベント由来の遷移時はこのオーバーレイは出さない。
  Issue の意図（「数分で次に行く」「続き見ることもできる」）に合致。

### 6-5. UA 指定

- 画面の `Settings` ドロワーで `Outbound User-Agent` を 1 つ入力できる。
- 値は `settings` テーブルの `outbound_user_agent` キーに保存。
- バックエンドのクロール処理が起動時／更新時に読み出して fetch のヘッダに使う。
- 「ブラウザ自体の UA を変える」は技術的に不可能なため、サーバー側のクロール時のみ反映される旨を
  注釈に明記する（要望者＝作者本人なので注釈で十分）。

## 7. ディレクトリ構成（実装着手時の想定）

```
nagara/                # ← 設計ドキュメント（このフォルダ）
nagara-app/            # ← 実装が始まったらここに置く想定
├ package.json
├ src/
│  ├ server/           # Hono ルート、SQLite 接続、クロールジョブ
│  │  ├ index.ts
│  │  ├ db.ts
│  │  ├ schema.sql
│  │  ├ crawler.ts
│  │  └ routes/
│  │     ├ channels.ts
│  │     ├ queue.ts
│  │     ├ episodes.ts
│  │     ├ history.ts
│  │     ├ settings.ts
│  │     └ filters.ts
│  └ web/              # Vite ビルド対象
│     ├ index.html
│     ├ main.tsx
│     ├ player/
│     │  ├ PlayerAdapter.ts
│     │  ├ YouTubeAdapter.ts
│     │  └ PodcastAudioAdapter.ts
│     ├ pages/
│     │  ├ Player.tsx
│     │  ├ Channels.tsx
│     │  └ History.tsx
│     └ components/...
├ data/                # SQLite ファイル（gitignore）
└ scripts/
   └ install-systemd.sh
```

実装フェーズに入った時の参考程度。本タスクの成果物は `nagara/` 以下のドキュメントのみ。

## 8. 実装着手順（受け入れ条件チェックリスト）

依頼の「ミニマムに作って触ってから設計を詰める」を満たすために、
**初回リリースの DoD** を以下とする:

- [ ] チャンネル URL を貼ると登録できる（YouTube・Podcast の両方）。
- [ ] 自動クロールが回り、`episodes` に新着が入る。
- [ ] プレイヤー画面で「Start」を押すと、キューの先頭から再生が始まる。
- [ ] エピソード末尾まで来たら自動で次に進む（`ended` 検知）。
- [ ] ザッピング設定時間が経つと「続けるか／切るか」のオーバーレイが出る。
- [ ] エピソードは履歴 `completed` で同じものが再キューされない。
- [ ] エピソード一覧から個別に Disable できる。
- [ ] Linux PC で `node dist/server.js` を systemd に登録するだけで常駐できる。

これが満たせれば、初回の "触ってみる" は十分にできる。
そこから実運用してみての違和感を [04-roadmap.md](./04-roadmap.md) のロードマップに沿って潰していく。
