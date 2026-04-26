# 03. MVP 設計

「**まずミニマムに作ってみてから考える**」「**自動キュー化より先に手動キューを作る**」という
依頼方針に従い、本書は MVP として実装すべき最小構成だけを書く。

## 1. コンセプト

- **「次これ見よう」をプラットフォーム横断で 1 本のキューに積み、頭から消化する**ツール。
- 入口は **URL 1 本貼って Enter**。チャンネル登録・自動クロール・おすすめは MVP では作らない。
- 再生方式は対象に応じて 3 通り（後述）。すべて 1 つの統一 UI から操作する。

## 2. 全体像

```
┌──────────── Linux PC（自宅 LAN 内、常駐） ────────────┐
│                                                       │
│  ┌──────────────┐    ┌────────────────────────┐       │
│  │ nagara API   │←──→│ better-sqlite3 (file) │       │
│  │ (Hono/Node)  │    │ ./data/nagara.db       │       │
│  └──────┬───────┘    └────────────────────────┘       │
│         │ Static                                       │
│         │ files     fetch (oEmbed, og:tags, RSS)       │
│  ┌──────▼───────┐                                      │
│  │ nagara Web   │ ──▲                                  │
│  │ (静的 SPA)   │   │                                  │
│  └──────────────┘   ▼                                  │
└─────────────────────┬──────────────────────────────────┘
                      │ HTTP (LAN)
               ┌──────▼───────┐
               │ Windows PC   │ ブラウザで http://nagara.local:<port>
               │ (Chrome 等)  │
               └──┬───────────┘
                  │
                  ▼
       ┌──────────────────────────────────────────┐
       │ 各サービス（YouTube は iframe 直接再生／  │
       │ TVer 等は新規タブ）                       │
       └──────────────────────────────────────────┘
```

ポイント:
- メディアのストリーム自体はサーバを経由しない。ブラウザから配信元へ直接。
- バックエンドの外部通信は **メタ取得（oEmbed / og:タグ / Podcast RSS）のみ**。

## 3. デプロイ戦略

要件「起動が面倒なのは避けたい」を最優先する。

### 採用: Linux PC で systemd 常駐

- `node dist/server.js` を `nagara.service` として登録。
- フロントとバックエンドは **同一プロセス／同一ポート** で配信（Hono が静的ファイルもサーブ）。
  → CORS 設定不要、起動コマンドが 1 個になる。
- ブラウザは `http://<linux-ip>:<port>` を開きっぱなしにしておけば再起動なしで使える。

### 不採用: Windows PC で常駐／Docker Compose

理由は前回の設計と同じ（タスクトレイ常駐の作り込み・Docker Engine 前提が「起動が面倒」要件に逆行）。

## 4. 採用技術スタック

「ミニマム」と「ローカル単体起動」を最優先。

| レイヤ | 採用 | 不採用と理由 |
|---|---|---|
| バックエンドランタイム | Node.js 22 LTS + TypeScript | — |
| バックエンドフレームワーク | **Hono**（Node アダプタ） | 静的配信＋ルーティング＋型推論が短く書ける |
| DB | **`better-sqlite3`**（同期ドライバ） | ORM は MVP では過剰 |
| マイグレーション | 起動時に `CREATE TABLE IF NOT EXISTS` を流すだけ | drizzle-kit/prisma migrate は不要 |
| RSS / HTML パース | `fast-xml-parser` ＋ 正規表現での meta タグ抽出 | jsdom は重い |
| フロントビルド | **Vite + React + TypeScript** | Next.js は SSR 不要なので過剰 |
| 状態管理 | `useState` / `useReducer` のみ | Zustand/Redux は不要 |
| HTTP クライアント | 素の `fetch` | SWR/React Query は不要 |
| スタイル | CSS Modules（ベタ書き） | Tailwind/shadcn は導入コスト > 効用 |

## 5. データモデル

SQLite。テーブル 4 個（前回の 5 個から `channels` を削除して `queue_items` を追加）。

```sql
-- アイテム（キューに入れた／入っていた 1 件）
-- 同じ URL を再追加することは可能（手動キューの尊重）。
CREATE TABLE IF NOT EXISTS items (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  source_url    TEXT NOT NULL,                    -- ユーザが貼った原 URL
  source_kind   TEXT NOT NULL CHECK (source_kind IN
                  ('youtube_video', 'audio_file', 'external_link')),
  external_id   TEXT,                              -- youtube_video: videoId / audio_file: NULL / external_link: NULL
  media_url     TEXT,                              -- audio_file: 直リン URL / 他: NULL
  title         TEXT,                              -- メタ取得後に埋まる
  author        TEXT,                              -- 投稿者名／チャンネル名／サイト名
  thumbnail_url TEXT,
  duration_seconds INTEGER,
  meta_status   TEXT NOT NULL DEFAULT 'pending'    -- 'pending' | 'ok' | 'failed'
                  CHECK (meta_status IN ('pending', 'ok', 'failed')),
  meta_error    TEXT,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

-- キュー（順序付き、いま再生中もこのテーブルで表現）
CREATE TABLE IF NOT EXISTS queue_items (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  item_id     INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
  position    INTEGER NOT NULL,
  status      TEXT NOT NULL DEFAULT 'queued'
                CHECK (status IN ('queued', 'playing')),
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_queue_position ON queue_items(position);
CREATE UNIQUE INDEX IF NOT EXISTS uq_queue_one_playing
  ON queue_items(status) WHERE status = 'playing';

-- 履歴（キューから消えた／流し終わった行を貯める）
CREATE TABLE IF NOT EXISTS history (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  item_id          INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
  status           TEXT NOT NULL CHECK (status IN ('completed', 'skipped', 'removed')),
  progress_seconds INTEGER,
  played_at        TEXT NOT NULL DEFAULT (datetime('now')),
  finished_at      TEXT
);
CREATE INDEX IF NOT EXISTS idx_history_item ON history(item_id);

-- KV 設定（ザッピング間隔等）
CREATE TABLE IF NOT EXISTS settings (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

設計上のメモ:

- `items` と `queue_items` を分離した理由は **「履歴に残った行を再キューしたい」** から。
  キューから消えた瞬間に `items` ごと削除すると、履歴・再追加で URL のメタを再取得する羽目になる。
- `queue_items.status = 'playing'` を最大 1 件に制限する部分インデックス（SQLite 3.8.0+）で
  「いま再生中はキュー全体で 1 件まで」を DB レベルで保証する。
  → アプリ側のロックを書かずに済む。
- 同じ `item_id` をキューに複数回入れることは許容（同じ URL を 2 回流したい用途のため）。
- `position` の連番管理は「常にゼロ詰めで再採番」ではなく **疎な整数で運用** し、
  「2 と 3 の間に挿入したいときは 2.5 → 整数化を後でやる」みたいなことはしない。
  挿入のたびに `UPDATE queue_items SET position = position + 1 WHERE position >= ?` でずらす
  シンプルな方式で十分（数十〜数百件しか並ばない想定なので速度問題は起きない）。

## 6. バックエンド API

すべて `application/json`。エラーは HTTP ステータス + `{ "error": "..." }`。
認証なし（LAN 限定）。

### 6-1. アイテム（メタ取得）

| Method | Path | 振る舞い |
|---|---|---|
| POST | `/api/items` | `{ url }` を受け取って `items` に保存し、メタ取得をキック |
| GET  | `/api/items/:id` | アイテム 1 件取得（メタ取得状態の確認用） |
| POST | `/api/items/:id/refetch` | メタ取得を再試行 |

`POST /api/items` の処理フロー:

```
1. URL を正規化（fragment 除去等）
2. URL から source_kind を判定
   ├ youtube.com / youtu.be / shorts / live → 'youtube_video' (videoId 抽出)
   ├ Content-Type が audio/* または拡張子 .mp3 .m4a .ogg .aac → 'audio_file' (HEAD で判定)
   └ それ以外                                              → 'external_link'
3. items に INSERT (meta_status='pending')
4. メタ取得を非同期で実行（プロセス内のキュー、せいぜい 4 並列）
   ├ youtube_video → oEmbed (https://www.youtube.com/oembed?url=...)
   ├ audio_file    → HEAD で Content-Length, ファイル名から title 推定
   └ external_link → HTML を GET して og:* を正規表現で抽出
5. items を UPDATE (meta_status='ok' or 'failed', title/author/thumbnail/duration をセット)
```

メタ取得は失敗してもキュー再生は破綻しない（再生は URL 直で出来るので、見出しが空のまま再生される）。

### 6-2. キュー操作（コア）

| Method | Path | 振る舞い |
|---|---|---|
| GET    | `/api/queue` | 現在のキューを `position` 昇順で返す（`status` 含む） |
| POST   | `/api/queue` | `{ url }` でキュー末尾に追加（内部で `POST /api/items` と同等の処理＋キュー追加） |
| POST   | `/api/queue/insert-next` | `{ url }`：いま再生中の **次** に挿入 |
| PATCH  | `/api/queue/:id` | `{ position }` の差し替え（並び替え） |
| DELETE | `/api/queue/:id` | キューから削除（履歴 status='removed' を 1 行追加） |
| POST   | `/api/queue/play` | `{ queueItemId }` を `status='playing'` に。直前の `playing` は完了として履歴へ |
| POST   | `/api/queue/complete` | `{ progressSeconds, status }` を受けて、いま `playing` の行を履歴に移して削除 |

`POST /api/queue/play` の動作:
- 直前の `playing` 行があれば、それを **`history`（status='skipped'）に移して削除**。
- 指定 `queueItemId` を `status='playing'` に更新。
- レスポンスとして「いま再生中のアイテムの全情報」と「次の queue_item」を返す。

`POST /api/queue/complete` の動作:
- `body.status` は `'completed' | 'skipped'`。
- `playing` の行を見つけて `history` に INSERT、`queue_items` から削除。
- 次のキュー先頭を返す（あれば）。

### 6-3. 履歴

| Method | Path | 振る舞い |
|---|---|---|
| GET  | `/api/history?limit=100` | 最新の履歴 |
| POST | `/api/history/:id/requeue` | 履歴の行をキュー末尾に再投入（同じ `item_id` で `queue_items` を作る） |

### 6-4. 設定

| Method | Path | 振る舞い |
|---|---|---|
| GET    | `/api/settings` | KV 全件 |
| PATCH  | `/api/settings` | KV 部分更新 |

主要キー:
| key | 型 | デフォルト | 説明 |
|---|---|---|---|
| `zapping_timeout_seconds` | number | `0`（無効） | エピソード再生開始から自動スキップまでの秒数 |
| `zapping_grace_seconds` | number | `10` | 「続き見る？」表示時間 |
| `default_playback_rate` | number | `1.0` | 再生速度デフォルト |
| `outbound_user_agent` | string | 空 | サーバが外部メタ取得時に使う UA（環境変数 `OUTBOUND_USER_AGENT` でも可） |

## 7. フロントエンド設計

画面は最小 3 つ（前回より 1 つ減らした）。

```
/                ← プレイヤー（メイン画面、キューも横に並ぶ）
/history         ← 履歴（読み取り＋再投入）
/settings        ← 設定（ザッピング・UA 等）
```

「チャンネル一覧」画面は MVP では存在しない（チャンネル管理機能が無いため）。

### 7-1. メインプレイヤー画面

```
┌────────────────────────────────────────────────────────────┐
│ nagara                              [履歴] [設定]            │
├────────────────────────────┬───────────────────────────────┤
│                             │ ▼ 次の再生（キュー）            │
│   ┌─────────────────────┐   │ ┌─────────────────────────┐  │
│   │ Player 領域          │   │ │ ☰ 1. タイトル A    ✕  │  │
│   │  - youtube_video →  │   │ │   チャンネル / 12:34    │  │
│   │    YT iframe        │   │ ├─────────────────────────┤  │
│   │  - audio_file →     │   │ │ ☰ 2. タイトル B    ✕  │  │
│   │    <audio>          │   │ │   投稿者 / 1:08:22      │  │
│   │  - external_link →  │   │ ├─────────────────────────┤  │
│   │    "外部で開く" UI  │   │ │ ☰ 3. ...               │  │
│   └─────────────────────┘   │ └─────────────────────────┘  │
│                             │                               │
│   タイトル / 投稿者 / 公開日 │   ─ URL を貼ってキュー追加 ─  │
│                             │   ┌─────────────────────┐    │
│   [▶/⏸] [⏭次へ] [×1.0 ▾]   │   │ https://...        │    │
│   [全画面] [外部で開く]     │   └────────────[追加]┘     │
└────────────────────────────┴───────────────────────────────┘
```

主要な UI 要素:
- 右ペイン上部に **キュー**（ドラッグで並び替え／× で削除）。
- 右ペイン下部に **URL 入力欄**。Enter または「追加」で末尾追加。
  - 入力値が複数行なら 1 行ずつまとめて追加（コピペで複数 URL 投入する用途）。
- いま再生中のアイテムは右ペインの **キュー 0 行目** として強調表示する
  （`queue_items.status='playing'` の行）。
- 「次これ再生」アクションはコンテキストメニュー（右クリック）or 各行のメニューから。

### 7-2. PlayerAdapter インターフェイス

「ドメインごとにロード／再生／一時停止／見終わり／速度／フルスクリーンを統一」する要件への
唯一の抽象化レイヤー。MVP では実装 3 つ。

```ts
type PlayerEvent =
  | { type: "ended" }
  | { type: "error"; code: string; message: string }
  | { type: "needs_user_action"; reason: "external_link" | "autoplay_blocked" };

interface PlayerAdapter {
  mount(container: HTMLElement): Promise<void>;
  load(item: Item): Promise<void>;
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
| `YouTubeAdapter` | `<div>` に YouTube IFrame Player API で `YT.Player` を生成。`onStateChange` の `ENDED` を `ended` に。`onError` 101/150 の場合は **`needs_user_action`（reason='external_link'）** を発火し、上位レイヤが「外部で開く」モードへ落とす |
| `AudioAdapter` | `<audio>` を生成。`ended`／`error` を変換。`playbackRate` で速度。フルスクリーンは無効 |
| `ExternalLinkAdapter` | プレイヤー領域に「外部で視聴中」UI を出す。`load()` 時に `window.open(url, '_blank')` を発火（ユーザ操作ハンドラ内で）。`getCurrentTime`/`getDuration` は `0`／`NaN` で返す。`ended` はユーザが「見終わった」を押した時に発火 |

### 7-3. 自動再生の制御

- 初回起動時はプレイヤー領域に **「再生を開始」ボタン**。クリックで `play()`。
- 連鎖再生は同一ジェスチャ文脈内で動くので追加対応不要。
- `play()` が `NotAllowedError` で reject されたら「Resume」ボタンを再表示する。

### 7-4. ザッピング機能

- 設定 `zapping_timeout_seconds`（デフォルト `0` = 無効）。
- 0 でない場合、エピソード再生開始から該当秒数経過時にオーバーレイ:

  ```
  ┌────────────────────────────┐
  │ そろそろ次に行きます       │
  │ 8 秒後に切り替え            │
  │ [このまま続ける] [いま切替] │
  └────────────────────────────┘
  ```

- 猶予秒数 `zapping_grace_seconds`（デフォルト 10）が経つか「いま切替」で
  `POST /api/queue/complete { status: 'skipped', progressSeconds }` を発行して次へ。
- 「このまま続ける」でタイマーリセット。
- `ended` 由来の遷移時はオーバーレイなしで即次へ（自然に終わったケース）。

**MVP のデフォルトを 0（無効）にする理由:**
- タスクフィードバック「自分で積んだものを強制スキップしたくない」を尊重。
- ユーザが運用してみて「やっぱりタイマー欲しい」となったら設定で有効化する。

### 7-5. URL 投入 → 再生までのフロント側フロー

```
URLを貼る → POST /api/queue { url }
            └→ items + queue_items 作成、メタ取得は非同期
キュー再取得 → 末尾に新しい行が追加される（title が空でも先に position は確定）
ポーリング or SWR で
items.meta_status='ok' を検知 → 該当行のタイトル等を更新表示
```

メタ取得が失敗しても `source_url` は分かるので、行は表示し続ける（タイトル欄が空 or "（メタ取得失敗）"）。

## 8. ディレクトリ構成（実装着手時の想定）

```
nagara/                # ← 設計ドキュメント（このフォルダ）
nagara-app/            # ← 実装が始まったらここに置く想定
├ package.json
├ src/
│  ├ server/
│  │  ├ index.ts
│  │  ├ db.ts
│  │  ├ schema.sql
│  │  ├ meta/                  # メタ取得（oEmbed / og:tags / RSS）
│  │  │  ├ youtube-oembed.ts
│  │  │  ├ og-tags.ts
│  │  │  └ audio-head.ts
│  │  └ routes/
│  │     ├ items.ts
│  │     ├ queue.ts
│  │     ├ history.ts
│  │     └ settings.ts
│  └ web/
│     ├ index.html
│     ├ main.tsx
│     ├ player/
│     │  ├ PlayerAdapter.ts
│     │  ├ YouTubeAdapter.ts
│     │  ├ AudioAdapter.ts
│     │  └ ExternalLinkAdapter.ts
│     ├ pages/
│     │  ├ Player.tsx
│     │  ├ History.tsx
│     │  └ Settings.tsx
│     └ components/...
├ data/                # SQLite ファイル（gitignore）
└ scripts/
   └ install-systemd.sh
```

## 9. 受け入れ条件（DoD）

依頼方針「ミニマムに作って触ってから設計を詰める」を満たすため、初回リリースの DoD:

- [ ] YouTube 動画 URL を貼ると、メタが自動で埋まってキュー末尾に入る。
- [ ] mp3 直リンを貼ると、メタが推定で埋まってキュー末尾に入り、`<audio>` で再生される。
- [ ] TVer のような任意の URL を貼ると、og:タグからメタが入り、キューに混ざる。
      再生は「外部で開く」となり、ユーザが「見終わった」を押せば次へ進む。
- [ ] キューはドラッグで並び替えできる。削除も「次これ再生」も動く。
- [ ] エピソード末尾まで来たら自動で次に進む（YouTube の `ended` / `<audio>` の `ended`）。
- [ ] ブラウザ再起動してもキューが復元される（SQLite 永続化）。
- [ ] 履歴画面で過去のアイテムを再投入できる。
- [ ] ザッピングタイマーは設定で有効化したときのみ動作する。
- [ ] Linux PC で `node dist/server.js` を systemd に登録するだけで常駐できる。

ここまで満たせば「触ってみる」が成立する。
そこから運用してみて、自動キューイングや投稿者管理の必要性を見極めて
[04-roadmap.md](./04-roadmap.md) のロードマップに沿って積み増す。
