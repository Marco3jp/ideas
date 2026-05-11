# 03. MVP 設計

「ミニマムに作って触ってから本設計に落とす」方針に従い、本書は MVP として実装すべき
最小構成だけを書く。要件は [01-requirements.md](./01-requirements.md)、
技術判断の根拠は [02-research.md](./02-research.md) を参照。

## 1. コンセプト

- **「次これ見よう」をプラットフォーム横断で 1 本のキューに積み、頭から消化する**ツール。
- 入口は **URL 1 本貼って Enter**。投入用に **メイン UI** と **スマホ向け補助 UI** の 2 系統を持ち、
  状態は WebSocket でリアルタイム同期する。
- 「ながら聞き」を成立させるため、**エピソードの切り替わりに人間操作を要求しない** ことが必達制約。
  再生経路は 3 つの Tier から URL に応じて自動選択する。
- **割り込み再生**（スタック復帰）で、いま見たいものを差し込んでも元の動画位置から戻れる。

## 2. 全体像

```
┌──────────── Linux PC（自宅 LAN 内、常駐） ────────────────┐
│                                                          │
│  ┌──────────────────────────────────────┐                │
│  │ nagara (Go 単一バイナリ)             │                │
│  │  ・REST API（Chi）                   │                │
│  │  ・WebSocket（coder/websocket）      │ ←─→ SQLite     │
│  │  ・HLS プロキシ                       │     ファイル   │
│  │  ・yt-dlp 子プロセス制御              │                │
│  │  ・Vite ビルド済みフロントを embed.FS │                │
│  │    で同梱して静的配信                  │                │
│  └──────────────────────────────────────┘                │
│            ▲                  ▲                          │
│            │ HTTP/WS          │ SSE/HTTP                 │
└────────────┼──────────────────┼──────────────────────────┘
             │                  │
   ┌─────────┴─────────┐    ┌───┴──────────────────┐
   │ Windows PC        │    │ スマホ                │
   │ Chrome（メイン UI）│    │ Chrome（補助 UI /quick）│
   │ + nagara 拡張     │    └──────────────────────┘
   └───────┬───────────┘
           │
           ▼
 ┌────────────────────────────────────────────────┐
 │ Tier 1: YouTube IFrame / <audio>               │
 │ Tier 2: <video> + hls.js（バックエンドプロキシ越し）│
 │ Tier 3: 拡張が制御する別タブ                    │
 └────────────────────────────────────────────────┘
```

ポイント:

- 配布物は **Go バイナリ 1 個と SQLite ファイル 1 個**。
- フロント（メイン UI と補助 UI）は同一 SPA。`/` がメイン、`/quick` が補助。
- Tier 1／Tier 3 のメディアストリームはサーバを経由しない。
- **Tier 2 のみバックエンドが HLS プロキシを兼ねる**（CORS／トークン期限を一箇所で吸収）。
- バックエンドの外部通信は「メタ取得」「`yt-dlp` 実行」「HLS プロキシの中継」の 3 系統。

## 3. デプロイ戦略

要件「起動が面倒なのは避けたい」を最優先する。

### 採用: Linux PC で systemd 常駐

- `go build` で出した **単一バイナリ**を `/usr/local/bin/nagara` に置く。
- `nagara.service` の `ExecStart=/usr/local/bin/nagara` で常駐。
- フロントは `embed.FS` でバイナリに同梱しているため、別途配置不要。
- ブラウザは `http://<linux-ip>:<port>` を開きっぱなしにしておけば再起動なしで使える。

### 不採用: Docker Compose

- 単一バイナリで完結する構成に対して、Docker Engine の前提は重い。
- 「起動が面倒」要件に逆行する。

## 4. 採用技術スタック

「ミニマム」「ローカル単体起動」「単一バイナリ配布」を最優先。

| レイヤ | 採用 | 補足 |
|---|---|---|
| バックエンド言語 | **Go** | バイナリ 1 個で配布。理由は [02-research.md](./02-research.md) §6-4 |
| ルーティング | **`github.com/go-chi/chi/v5`** | net/http に直接乗る軽量ルータ |
| WebSocket | **`github.com/coder/websocket`** | Go 作者陣推奨、メンテ活発 |
| DB | **`modernc.org/sqlite`** | pure Go、CGo 不要。WAL モードで運用 |
| 静的ファイル同梱 | **標準 `embed.FS`** | Vite の `dist/` を同梱 |
| 子プロセス（yt-dlp） | **`os/exec`** | 標準 |
| HLS プロキシ | **`net/http` + `io.Copy`** | マニフェスト書換は自前実装 |
| マイグレーション | 起動時に `CREATE TABLE IF NOT EXISTS` を流す | drizzle-kit 等は使わない |
| RSS / HTML パース | 標準 `encoding/xml` ＋ 正規表現での meta タグ抽出 | DOM パーサは使わない |
| フロントビルド | **Vite + React + TypeScript** | メイン UI と補助 UI を 1 つの SPA で配信 |
| HLS 再生 | **`hls.js`**（Tier 2 専用） | Chrome は HLS をネイティブで `<video>` に流せないため必須 |
| Chrome 拡張 | **MV3、TypeScript で書く小さなモノレポ** | 配布は MVP では開発者モード読込み |
| 状態管理 | `useState` / `useReducer` のみ。WebSocket は 1 つのコンテキストで配信 | Zustand/Redux は不要 |
| HTTP クライアント | 素の `fetch` | SWR/React Query は不要（WS で push されるため） |
| スタイル | CSS Modules | Tailwind/shadcn は導入コスト > 効用 |

## 5. データモデル

SQLite。テーブル 4 個。`source_kind` は **再生方式（Tier）** と 1 対 1 対応する。
`queue_items` は通常キューと割り込みスタックを 1 テーブルで表現する。

```sql
-- アイテム（キューに入れた／入っていた 1 件）
-- 同じ URL を再追加することは可能（手動キューの尊重）。
CREATE TABLE IF NOT EXISTS items (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  source_url    TEXT NOT NULL,                    -- ユーザが貼った原 URL
  source_kind   TEXT NOT NULL CHECK (source_kind IN
                  ('youtube_embed',     -- Tier 1: YouTube IFrame Player API
                   'audio_file',        -- Tier 1: <audio> 直接
                   'ytdlp_stream',      -- Tier 2: yt-dlp 抽出 → hls.js
                   'extension_tab')),   -- Tier 3: 拡張が別タブで再生
  external_id   TEXT,                              -- youtube_embed: videoId / 他: 任意
  media_url     TEXT,                              -- audio_file: 直リン URL / ytdlp_stream: マニフェスト URL（プロキシ前） / 他: NULL
  stream_kind   TEXT,                              -- ytdlp_stream のとき 'hls' | 'progressive'
  title         TEXT,
  author        TEXT,
  thumbnail_url TEXT,
  duration_seconds INTEGER,
  meta_status   TEXT NOT NULL DEFAULT 'pending'
                  CHECK (meta_status IN ('pending', 'ok', 'failed')),
  meta_error    TEXT,
  resolution_status TEXT NOT NULL DEFAULT 'pending'
                  CHECK (resolution_status IN ('pending', 'ok', 'failed', 'na')),
  resolved_at   TEXT,                              -- ytdlp_stream の URL は短期失効するので保存時刻も記録
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

-- キュー＋割り込みスタックを 1 テーブルで表現
CREATE TABLE IF NOT EXISTS queue_items (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  item_id           INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
  status            TEXT NOT NULL
                      CHECK (status IN ('queued', 'playing', 'paused_for_interrupt')),
  position          REAL,                          -- queued: 順序キー。playing/paused_for_interrupt: NULL
  paused_at_seconds INTEGER,                       -- paused_for_interrupt のとき、停止時点の再生位置（秒）
  interrupts_id     INTEGER REFERENCES queue_items(id) ON DELETE SET NULL,
                                                     -- 割り込み中アイテムが「誰を割り込んだか」を指す
  created_at        TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_queue_position ON queue_items(position);
CREATE INDEX IF NOT EXISTS idx_queue_interrupts ON queue_items(interrupts_id);
-- いま再生中は 1 件だけ
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

-- KV 設定
CREATE TABLE IF NOT EXISTS settings (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 5-1. 設計上のメモ

- `items` と `queue_items` を分離した理由は **「履歴に残った行を再キューしたい」** から。
  キューから消えた瞬間に `items` ごと削除すると、履歴・再追加で URL のメタを再取得する羽目になる。
- `queue_items.status='playing'` は部分ユニークインデックスで「全体で 1 件まで」を DB レベル保証。
- `position` は **疎な浮動小数（REAL）** で管理し、間に挿入する時は中央値を取って採番する
  （`(prev.position + next.position) / 2`）。これにより並び替え／先頭挿入で全行 UPDATE する必要がなくなる。
  値が極端に細かくなったら起動時にリバランスする（数十〜数百件想定なので頻度は低い）。
- `resolution_status` と `resolved_at` を持つ理由: yt-dlp 由来の m3u8 はトークン付きで
  数十分〜数時間で失効する。再生開始時に `resolved_at` の経過時間を見て、必要なら再解決する。

### 5-2. 割り込みスタックの表現

- 通常再生: 該当行 `status='playing'`, `position=NULL`, `interrupts_id=NULL`。
- 割り込み中: 直前まで再生中だった行を `status='paused_for_interrupt'` に更新し、
  `paused_at_seconds` に再生位置を保存。割り込みアイテムを新規作成し
  `status='playing'`, `interrupts_id=<割り込まれた側の id>` に。
- 多段の割り込み: 上記を再帰的に繰り返す。`interrupts_id` のチェーンがスタックを成す。
- 復帰: `playing` を `history` へ移して削除した後、`interrupts_id` が `NULL` でない場合は
  そのチェーンを辿って最も新しい `paused_for_interrupt` を `playing` に戻す。
  フロントには `paused_at_seconds` を返してシークさせる。

## 6. バックエンド API

すべて `application/json`（ストリーム系を除く）。エラーは HTTP ステータス + `{ "error": "..." }`。
認証なし（LAN 限定）。

### 6-1. アイテム（メタ取得）

| Method | Path | 振る舞い |
|---|---|---|
| POST | `/api/items` | `{ url }` を受け取って `items` に保存し、メタ取得をキック |
| GET  | `/api/items/:id` | アイテム 1 件取得（メタ取得状態の確認用） |
| POST | `/api/items/:id/refetch` | メタ取得を再試行 |
| POST | `/api/items/:id/resolve` | Tier 2 用の yt-dlp 再解決（短期失効した m3u8 用） |

`POST /api/items` の処理フロー:

```
1. URL を正規化（fragment 除去等）
2. URL から source_kind（採用 Tier）を判定
   ├ youtube.com / youtu.be / shorts / live    → 'youtube_embed'  (Tier 1)
   ├ Content-Type が audio/* または .mp3 .m4a .ogg .aac → 'audio_file' (Tier 1, HEAD で判定)
   ├ tver.jp / その他「埋め込み拒否＋DRM」既知ホスト → 'extension_tab' (Tier 3)
   └ それ以外                                  → 'ytdlp_stream'   (Tier 2 候補)
3. items に INSERT (meta_status='pending', resolution_status='pending'|'na')
4. メタ取得を goroutine で非同期実行（並列数は設定で制限）
   ├ youtube_embed  → oEmbed
   ├ audio_file     → HEAD で Content-Length, ファイル名から title 推定
   ├ ytdlp_stream   → yt-dlp -j で JSON 取得 → title/uploader/thumbnail を取り出す
   └ extension_tab  → og:* を正規表現で抽出（HTML を GET）
5. ytdlp_stream のときは続けて解決処理:
   ├ formats[] が空 / DRM-only / 音声のみ → resolution_status='failed' に落とし、
   │   source_kind を 'extension_tab' に降格（Tier 3 へフォールバック）
   ├ HLS が取れる → media_url=manifest_url, stream_kind='hls', resolution_status='ok'
   └ progressive が取れる → media_url=直リン, stream_kind='progressive', resolution_status='ok'
6. items を UPDATE → WebSocket で item_meta_updated を broadcast
```

メタ取得は失敗してもキュー再生は破綻しない（再生は URL 直で出来るので、見出しが空のまま再生される）。

### 6-2. キュー操作

| Method | Path | 振る舞い |
|---|---|---|
| GET    | `/api/queue` | 現在のキュー全体（通常キュー＋playing＋割り込みスタック）を返す |
| POST   | `/api/queue` | `{ url }` でキュー末尾に追加（内部で `POST /api/items` 相当の処理＋キュー追加） |
| POST   | `/api/queue/insert-next` | `{ url }`：いま再生中の次に来る位置に挿入（position を `playing` の論理直後に） |
| POST   | `/api/queue/play-now` | `{ url }`：今すぐ割り込みで見る（後述） |
| PATCH  | `/api/queue/:id` | `{ position }` を変更して並び替え |
| DELETE | `/api/queue/:id` | キューから削除（履歴 status='removed' を 1 行追加） |
| POST   | `/api/queue/skip` | いま再生中を `skipped`（progress_seconds 付き）として終了し次へ |
| POST   | `/api/queue/complete` | 自然終了による次への遷移。割り込みスタックがあれば復帰、無ければキュー先頭を pop |

すべての操作の結果は `queue_updated` イベントとして WebSocket で broadcast される。

`POST /api/queue/play-now` の動作:

```
1. items を作成（POST /api/items 相当）。
2. 現在 playing の queue_item Q があれば:
   ├ Q.status を 'paused_for_interrupt' に更新
   ├ body の paused_at_seconds（フロントから送られる再生位置）を保存
3. 新しい queue_item I を作成:
   ├ status = 'playing'
   ├ position = NULL
   ├ interrupts_id = Q.id（Q が無ければ NULL）
4. WebSocket で broadcast。
5. レスポンスとして再生に必要な情報（item と I）を返す。
```

`POST /api/queue/complete` の動作（自然終了による次へ）:

```
1. いま playing の queue_item X を history に移す（status='completed', progress_seconds=null）。
2. X を queue_items から削除。
3. X.interrupts_id が指す Y を SELECT。Y がいれば:
   ├ Y.status を 'paused_for_interrupt' から 'playing' に戻す。
   ├ Y.paused_at_seconds を返してフロントにシークさせる。
4. Y が無ければ通常キュー先頭（position 最小の queued）を 'playing' に昇格。
5. WebSocket で playback_changed と queue_updated を broadcast。
```

`POST /api/queue/skip` の動作（ユーザによる飛ばし）:

- `complete` とほぼ同じだが、history への `status` が `skipped`、
  body から `progress_seconds` を受け取って history と paused_at_seconds に反映。
- 割り込み中の skip は「割り込みアイテムを破棄して元に戻る」と同じ意味になる。

### 6-3. 履歴

| Method | Path | 振る舞い |
|---|---|---|
| GET  | `/api/history?limit=100` | 最新の履歴 |
| POST | `/api/history/:id/requeue` | 履歴の行をキュー末尾に再投入（同じ `item_id` で `queue_items` を作る） |

### 6-4. Tier 2 — HLS プロキシ

| Method | Path | 振る舞い |
|---|---|---|
| GET | `/api/stream/:itemId/manifest.m3u8` | items のマニフェストをバックエンドが取得して中継。`Access-Control-Allow-Origin: *` 付与、必要に応じてトークンも付与。マニフェスト本体は **セグメント URL を `/api/stream/:itemId/segment?u=<encoded>` に書き換えた**ものを返す |
| GET | `/api/stream/:itemId/segment` | 上記マニフェストから呼ばれるセグメントを中継 |

セキュリティ:

- `:itemId` で参照されたアイテム以外の URL は中継しない。
- セグメント URL は **マニフェスト解析時に DB 側で許可リストに追加**したホストのみ通す。
- これにより SSRF を構造的に防止する。

### 6-5. Tier 3 — Chrome 拡張連携

| Method | Path | 振る舞い |
|---|---|---|
| POST | `/api/extension/play` | 本体 → 拡張へ「このタブで再生開始してくれ」を投げる |
| POST | `/api/extension/event` | 拡張のサービスワーカーが本体へ送る通知（`ended` / `error` / `progress`） |

詳細フローは §7-4。

### 6-6. WebSocket

| Path | 振る舞い |
|---|---|
| `GET /api/ws` | WebSocket ハンドシェイク。接続後はサーバ → クライアントの片方向 push がメイン |

クライアントへ push されるイベント（type で識別）:

| `type` | payload | 発火タイミング |
|---|---|---|
| `queue_updated` | 現キュー全体のスナップショット | キューの追加・削除・並び替え・割り込み・復帰 |
| `item_meta_updated` | 該当 `item` の最新スナップショット | メタ取得完了／yt-dlp 解決完了／降格 |
| `playback_changed` | `{ playing: queueItem, paused_at_seconds: number? }` | 自然遷移／割り込み／復帰 |
| `extension_event` | `{ kind, itemId, ... }` | 拡張からの ended/error/progress を中継 |

クライアント → サーバの方向は基本使わない（REST API を叩く）。
ping/pong は coder/websocket の標準機能で扱う。

サーバ側の実装方針:

- 接続を保持する **Hub** を 1 つ持つ（goroutine + チャネル）。
- 状態変更を起こす全 API は、コミット後に `hub.broadcast(event)` を呼ぶ。
- 多数のクライアントは想定しない（同一 LAN の数台程度）が、ブロードキャスト時の
  チャネル詰まりは select + default-drop でクライアント側を切断する。

## 7. フロントエンド設計

ルートは 4 つ。すべて同一 SPA の中。

```
/                ← メインプレイヤー画面（プレイヤー＋キュー＋設定ドロワー）
/quick           ← 補助 UI（軽量・スマホ前提）
/history         ← 履歴（読み取り＋再投入）
/settings        ← 設定（UA 等）
```

### 7-1. メインプレイヤー画面

```
┌────────────────────────────────────────────────────────────┐
│ nagara                              [履歴] [設定]            │
├────────────────────────────┬───────────────────────────────┤
│                             │ ▼ 次の再生（キュー）            │
│   ┌─────────────────────┐   │ ┌─────────────────────────┐  │
│   │ Player 領域          │   │ │ ☰ 1. タイトル A    ✕  │  │
│   │  - youtube_embed →  │   │ │   チャンネル / 12:34    │  │
│   │    YT iframe        │   │ ├─────────────────────────┤  │
│   │  - audio_file →     │   │ │ ☰ 2. タイトル B    ✕  │  │
│   │    <audio>          │   │ │   投稿者 / 1:08:22      │  │
│   │  - ytdlp_stream →   │   │ ├─────────────────────────┤  │
│   │    <video>+hls.js   │   │ │ ☰ 3. ...               │  │
│   │  - extension_tab →  │   │ └─────────────────────────┘  │
│   │    "別タブ再生中"   │   │                               │
│   └─────────────────────┘   │                               │
│                             │                               │
│   タイトル / 投稿者 / 公開日 │   ─ URL を貼ってキュー追加 ─  │
│                             │   ┌─────────────────────┐    │
│   [▶/⏸] [⏭次へ] [×1.0 ▾]   │   │ https://...        │    │
│   [全画面] [今すぐ割り込み] │   └─────────[追加][割込]┘   │
└────────────────────────────┴───────────────────────────────┘
```

主要な UI 要素:

- 右ペイン上部に **キュー**（ドラッグで並び替え／× で削除）。
- 右ペイン下部に **URL 入力欄**。「追加」（末尾追加）と「割込」（今すぐ割り込み）が並ぶ。
- いま再生中のアイテムは右ペインの **キュー 0 行目** として強調表示する。
- **割り込み中**は、現在再生中の上に「←〇〇に戻る予定」の小さな表示を出す。
  スタック深度が 2 以上ある場合はチェーンを示す（「← A に戻る予定 ← B に戻る予定」）。
- WebSocket からのイベントで、キュー一覧／プレイヤー上部情報／割り込み表示が即時更新される。

### 7-2. 補助 UI（`/quick`）

スマホブラウザで快適に開けるシンプルなページ。

```
┌──────────────────────────┐
│ nagara quick add          │
├──────────────────────────┤
│ URL                       │
│ ┌────────────────────┐    │
│ │ https://...        │    │
│ └────────────────────┘    │
│                           │
│ [    末尾に追加    ]      │
│ [    次に再生      ]      │
│ [  今すぐ割り込み  ]      │
│                           │
│ 直近の追加（5件）          │
│  ・タイトル A             │
│  ・...                    │
└──────────────────────────┘
```

実装方針:

- 同一 SPA のサブルート（`/quick`）。React コンポーネントを 1 ファイルで完結させ、
  メイン UI と共有するコードは型・API クライアントだけにする。
- スマホからの操作を想定し、ボタンを十分に大きく、入力欄は OS のクリップボード貼り付けがしやすい配置に。
- 直近の追加履歴は WebSocket の `queue_updated` を購読して受け取る
  （補助 UI 側も最新状態を反映する）。
- 共有メニュー連携:
  - PWA 化（`manifest.webmanifest`）して、スマホのホーム画面に追加できるようにする。
  - **`/quick?url=...`** をクエリで受けて初期値に入れる
    → ブラウザの共有メニューから URL クエリつきで叩く形を実現できる。

### 7-3. PlayerAdapter インターフェイス

「ドメインごとに操作を統一」する唯一の抽象化レイヤー。MVP では実装 4 つ。

```ts
type PlayerEvent =
  | { type: "ended" }
  | { type: "error"; code: string; message: string }
  | { type: "fallback_required"; toTier: 2 | 3 }
  | { type: "needs_user_action"; reason: "autoplay_blocked" | "extension_missing" };

interface PlayerAdapter {
  mount(container: HTMLElement): Promise<void>;
  load(item: Item, startAtSeconds?: number): Promise<void>;
  play(): Promise<void>;
  pause(): Promise<void>;
  setRate(rate: number): Promise<void>;
  getCurrentTime(): number;
  getDuration(): number;
  on(handler: (e: PlayerEvent) => void): void;
  destroy(): void;
}
```

`load(item, startAtSeconds)` の `startAtSeconds` は **割り込み復帰時のシーク**に使う。

| 実装 | Tier | 中身 |
|---|---|---|
| `YouTubeEmbedAdapter` | 1 | YouTube IFrame Player API。`onStateChange` の `ENDED` を `ended` に。`onError` 101/150 で `fallback_required(toTier=2)` を発火。シーク復帰は `seekTo(startAtSeconds, true)` |
| `AudioFileAdapter` | 1 | `<audio>` を生成。`ended`／`error` を変換。シーク復帰は `audio.currentTime = startAtSeconds` |
| `HlsVideoAdapter` | 2 | `<video>` を生成し、`hls.js` で `/api/stream/:id/manifest.m3u8` をロード。`Hls.Events.MANIFEST_PARSED` の後にシーク。`ended`/`error` をそのまま転送 |
| `ExtensionTabAdapter` | 3 | 拡張インストール状態を確認。未インストールなら `needs_user_action(extension_missing)` を発火。インストール済みなら `POST /api/extension/play` を投げ、WebSocket の `extension_event` を待ち受ける |

### 7-4. Tier 3（Chrome 拡張モード）の往復シーケンス

```
[本体フロント]                                       [本体バックエンド]
     │ POST /api/extension/play  { itemId, sourceUrl }      │
     ├─────────────────────────────────────────────────────►│
     │                                                       │
     │ ◄── chrome.runtime.sendMessage（externally_connectable）
     │                                                       │
[Chrome 拡張 service worker]                                  │
     │ chrome.tabs.create or chrome.tabs.update              │
     │   ({ url: sourceUrl })                                │
     │                                                       │
[コンテンツスクリプト（対象ドメインで動く）]                  │
     │ document.querySelector('video') を監視                │
     │ video.addEventListener('ended', ...)                  │
     │                                                       │
     │ ended 検知時:                                         │
     │ chrome.runtime.sendMessage  → service worker へ       │
     │                                                       │
     │ service worker: fetch('http://nagara.local/api/extension/event'
     │                       , { body:{ kind:'ended', itemId } })
     ├─────────────────────────────────────────────────────►│
     │                                                       │
     │ ◄─── WebSocket で extension_event を本体フロントへ ───│
     │                                                       │
     │ ExtensionTabAdapter.on() が ended を発火              │
     │ → POST /api/queue/complete                            │
```

重要ポイント:

- 拡張は **ユーザ自身がブラウザで普通に閲覧するのを自動化する** だけ。
  DRM 解除・ストリーム保存はしない。表示は当該サイトの公式プレイヤーが行う。
- `chrome.tabs.create` の戻り値 `tab.id` を保持し、次の Tier 3 アイテムでは
  **同じタブの URL を `chrome.tabs.update(tabId, { url })` で差し替える**
  （タブ増殖防止・autoplay 文脈の維持）。
- 拡張未インストール時は、Tier 3 アイテム再生開始時にインストール案内バナーを出す。

### 7-5. 自動再生制御

- 初回起動時はプレイヤー領域に **「再生を開始」ボタン**。クリックで `play()`。
- 連鎖再生は同一ジェスチャ文脈で動くので追加対応不要。
- `play()` が `NotAllowedError` で reject されたら「Resume」ボタンを再表示する。
- `<iframe>` 親要素には `allow="autoplay"` を付ける。

### 7-6. Tier 2 の m3u8 短期失効への対処

`HlsVideoAdapter.load()` 時、`item.resolved_at` をサーバから受け取ってフロントで判定:

```ts
const RESOLUTION_TTL = 30 * 60 * 1000; // 30 分（保守的に短め）
if (Date.now() - new Date(item.resolved_at).getTime() > RESOLUTION_TTL) {
  await fetch(`/api/items/${item.id}/resolve`, { method: 'POST' });
  // WebSocket の item_meta_updated を待って最新 item を使う
}
```

TTL は設定可能（`settings.ytdlp_resolution_ttl_seconds`）。
解決失敗（DRM 等）が後から発覚したら、サーバ側で `source_kind` を `extension_tab` に降格させる。

## 8. ディレクトリ構成（実装着手時の想定）

```
nagara/                # ← 設計ドキュメント（このフォルダ）
nagara-app/            # ← 実装本体（モノレポ）
├ go.mod
├ go.sum
├ cmd/
│  └ nagara/
│     └ main.go               # エントリポイント
├ internal/
│  ├ server/                  # HTTP / WebSocket / 静的配信
│  │  ├ router.go
│  │  ├ hub.go                # WebSocket Hub
│  │  ├ static.go             # embed.FS の配信
│  │  └ handlers/
│  │     ├ items.go
│  │     ├ queue.go
│  │     ├ history.go
│  │     ├ settings.go
│  │     ├ stream.go          # HLS プロキシ
│  │     ├ extension.go
│  │     └ ws.go              # /api/ws
│  ├ store/                   # SQLite アクセス
│  │  ├ db.go
│  │  ├ schema.sql            // go:embed
│  │  ├ items.go
│  │  ├ queue.go
│  │  └ history.go
│  ├ meta/                    # メタ取得
│  │  ├ youtube_oembed.go
│  │  ├ og_tags.go
│  │  └ audio_head.go
│  ├ ytdlp/                   # Tier 2 解決
│  │  ├ resolve.go
│  │  └ classify.go           # DRM/HLS/progressive 判定
│  └ events/                  # イベント定義
│     └ events.go
├ web/                        # フロント（Vite + React + TS）
│  ├ index.html
│  ├ src/
│  │  ├ main.tsx
│  │  ├ player/
│  │  │  ├ PlayerAdapter.ts
│  │  │  ├ YouTubeEmbedAdapter.ts
│  │  │  ├ AudioFileAdapter.ts
│  │  │  ├ HlsVideoAdapter.ts
│  │  │  └ ExtensionTabAdapter.ts
│  │  ├ pages/
│  │  │  ├ Player.tsx
│  │  │  ├ Quick.tsx
│  │  │  ├ History.tsx
│  │  │  └ Settings.tsx
│  │  ├ ws/
│  │  │  └ NagaraSocket.ts    # WS 接続・購読
│  │  └ components/...
│  └ public/
│     └ manifest.webmanifest  # PWA 化（補助 UI 用）
├ extension/                  # Chrome 拡張（MV3）
│  ├ manifest.json
│  ├ service-worker.ts
│  └ content/
│     └ video-watcher.ts
├ data/                       # SQLite ファイル（gitignore）
└ scripts/
   ├ install-systemd.sh
   └ install-ytdlp.sh
```

ビルド:

- `pnpm --filter web build` で `web/dist/` を生成。
- Go から `//go:embed all:web/dist` で同梱。
- `go build -o nagara ./cmd/nagara` で完成。

## 9. 受け入れ条件（DoD）

初回リリースの DoD:

**コア体験**
- [ ] メイン UI と補助 UI（`/quick`）の両方から URL を貼って末尾追加できる。
- [ ] 補助 UI で追加した瞬間に、メイン UI のキュー末尾に行が現れる（WebSocket 反映）。
- [ ] メタ取得が完了するとタイトル等がメイン UI に自動で反映される。
- [ ] 「今すぐ割り込みで見る」を実行すると、現再生中の再生位置が保存される。
- [ ] 割り込みアイテムが終了すると、元のアイテムが保存位置から自動復帰する。
- [ ] 多段の割り込み（割り込み中に更に割り込み）でもスタックが正しく復帰する。
- [ ] キューはドラッグで並び替えできる。削除も「次に再生」も動く。
- [ ] ブラウザ再起動してもキュー・割り込みスタックが復元される（SQLite 永続化）。
- [ ] 履歴画面で過去のアイテムを再投入できる。

**Tier 1（ネイティブ埋め込み）**
- [ ] YouTube 動画 URL を貼ると、oEmbed でメタが埋まり、IFrame Player API で再生される。
- [ ] mp3 直リンを貼ると、メタが推定で埋まり、`<audio>` で再生される。
- [ ] エピソード末尾で自動で次に進む（人間操作なしで連続再生）。

**Tier 2（yt-dlp + hls.js）**
- [ ] yt-dlp 対応サイトの DRM 無し動画 URL を貼ると、バックエンドで m3u8 が解決される。
- [ ] HLS プロキシ経由で `hls.js` がアプリ内 `<video>` で再生する。
- [ ] `<video>` の `ended` イベントで次へ自動遷移する。
- [ ] 短期失効（30 分以上経過）時は再生開始時に自動で再解決される。
- [ ] DRM 検出で yt-dlp 解決が失敗したアイテムは Tier 3 に自動降格される。

**Tier 3（Chrome 拡張モード）**
- [ ] nagara 専用 Chrome 拡張（MV3）を開発者モードで読み込める。
- [ ] TVer の URL を貼ると拡張が新しいタブを開いて公式プレイヤーで再生する。
- [ ] そのタブで動画が `ended` した瞬間に拡張がイベントを本体に送り、本体が次のエピソードへ進める。
- [ ] 次のエピソードが Tier 3 のときは、同じタブの URL を更新して使い回す。
- [ ] 拡張未インストール時は、Tier 3 アイテム再生開始時にインストール案内バナーが出る。

**運用**
- [ ] Linux PC で `go build` した単一バイナリを systemd で起動するだけで使い始められる。
- [ ] yt-dlp バイナリは `scripts/install-ytdlp.sh` で取得・更新できる。
- [ ] 補助 UI は PWA 対応で、スマホのホーム画面に追加できる。
