# 03. MVP 設計

「**まずミニマムに作ってみてから考える**」「**自動キュー化より先に手動キューを作る**」
「**「ながら聞き」コンセプトを破綻させない**」という依頼方針に従い、
本書は MVP として実装すべき最小構成だけを書く。

## 1. コンセプト

- **「次これ見よう」をプラットフォーム横断で 1 本のキューに積み、頭から消化する**ツール。
- 入口は **URL 1 本貼って Enter**。チャンネル登録・自動クロール・おすすめは MVP では作らない。
- ながら聞きを成立させるため、**エピソードの切り替わりに人間操作を要求しない** ことが必達制約。
- 再生経路は **3 つの Tier**（Tier 1: ネイティブ埋め込み／Tier 2: yt-dlp+hls.js／
  Tier 3: Chrome 拡張モード）から、URL に応じて自動選択。
  「外部タブ＋手動進行」（旧版にあった Tier 4 相当）は **コンセプトを破綻させるので採用しない**。

## 2. 全体像

```
┌──────────── Linux PC（自宅 LAN 内、常駐） ────────────┐
│                                                       │
│  ┌──────────────┐    ┌─────────────────────────┐      │
│  │ nagara API   │←──→│ better-sqlite3 (file)  │      │
│  │ (Hono/Node)  │    │ ./data/nagara.db        │      │
│  │              │    └─────────────────────────┘      │
│  │  ・メタ取得（oEmbed / og:tags / RSS）              │
│  │  ・yt-dlp 子プロセス起動 → m3u8 解決               │
│  │  ・HLS プロキシ（CORS 解消とトークン付与）         │
│  │  ・拡張からの ended 通知の受け口（SSE/WS）         │
│  └──────┬───────┘                                     │
│         │ Static files                                │
│  ┌──────▼───────┐                                     │
│  │ nagara Web   │                                     │
│  │ (静的 SPA)   │                                     │
│  └──────────────┘                                     │
└─────────────────────┬─────────────────────────────────┘
                      │ HTTP (LAN)
               ┌──────▼───────┐
               │ Windows PC   │ Chrome で http://nagara.local:<port>
               │ (Chrome)     │ ＋ nagara Chrome 拡張（Tier 3 用）
               └──┬───────────┘
                  │
                  ▼
   ┌─────────────────────────────────────────────────────────┐
   │ Tier 1: YouTube IFrame Player API / <audio>             │
   │ Tier 2: <video> + hls.js（バックエンドプロキシ越し）    │
   │ Tier 3: 拡張が制御する別タブ（YouTube 拒否動画 / TVer 等)│
   └─────────────────────────────────────────────────────────┘
```

ポイント:
- Tier 1／Tier 3 のメディアストリームはサーバを経由しない。
- **Tier 2 のみバックエンドが HLS プロキシを兼ねる**（CORS／トークン期限の問題を一箇所で吸収）。
- バックエンドの外部通信は「メタ取得」「`yt-dlp -j ...` 実行」「HLS プロキシの中継」の 3 系統。

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

「ミニマム」と「ローカル単体起動」を最優先。Tier 2／Tier 3 を成立させるための追加要素も明示。

| レイヤ | 採用 | 不採用と理由 |
|---|---|---|
| バックエンドランタイム | Node.js 22 LTS + TypeScript | — |
| バックエンドフレームワーク | **Hono**（Node アダプタ） | 静的配信＋ルーティング＋型推論が短く書ける |
| DB | **`better-sqlite3`**（同期ドライバ） | ORM は MVP では過剰 |
| マイグレーション | 起動時に `CREATE TABLE IF NOT EXISTS` を流すだけ | drizzle-kit/prisma migrate は不要 |
| RSS / HTML パース | `fast-xml-parser` ＋ 正規表現での meta タグ抽出 | jsdom は重い |
| 動画ストリーム解決（Tier 2） | **`yt-dlp` バイナリを子プロセスで `-j` 実行** | サーバー側に同梱（`scripts/install.sh` で自動取得）。Node ライブラリのラッパは挙動が枯れていない |
| HLS プロキシ（Tier 2） | Hono の素のルートで `fetch` → ストリーム転送 | 専用プロキシサーバを立てる必要なし |
| 本体 ↔ 拡張通信（Tier 3 / 本体 → 拡張） | **`externally_connectable` + `chrome.runtime.sendMessage`** | postMessage より素直に MV3 で動く |
| 拡張 → 本体通信（Tier 3 / イベント通知） | **SSE（`text/event-stream`）** | ブラウザ側の追加ライブラリ不要。WebSocket は MVP には重い |
| フロントビルド | **Vite + React + TypeScript** | Next.js は SSR 不要なので過剰 |
| HLS 再生 | **`hls.js`**（Tier 2 専用） | Chrome は HLS をネイティブで `<video>` に流せないため必須 |
| Chrome 拡張 | **MV3、TypeScript で書く小さなモノレポ** | 配布は MVP では開発者モード読込み。CWS 公開は将来 |
| 状態管理 | `useState` / `useReducer` のみ | Zustand/Redux は不要 |
| HTTP クライアント | 素の `fetch` | SWR/React Query は不要 |
| スタイル | CSS Modules（ベタ書き） | Tailwind/shadcn は導入コスト > 効用 |

## 5. データモデル

SQLite。テーブル 4 個。`source_kind` は **再生方式（Tier）** とほぼ 1 対 1 対応する。

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
  title         TEXT,                              -- メタ取得後に埋まる
  author        TEXT,                              -- 投稿者名／チャンネル名／サイト名
  thumbnail_url TEXT,
  duration_seconds INTEGER,
  meta_status   TEXT NOT NULL DEFAULT 'pending'    -- 'pending' | 'ok' | 'failed'
                  CHECK (meta_status IN ('pending', 'ok', 'failed')),
  meta_error    TEXT,
  resolution_status TEXT NOT NULL DEFAULT 'pending' -- ytdlp_stream の m3u8 解決状態
                  CHECK (resolution_status IN ('pending', 'ok', 'failed', 'na')),
  resolved_at   TEXT,                              -- ytdlp_stream の URL は短期失効するので保存時刻も記録
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
- `resolution_status` と `resolved_at` を持つ理由: yt-dlp 由来の m3u8 はトークン付きで
  数十分〜数時間で失効する。再生開始時に `resolved_at` から経過時間が一定以上なら
  「再解決」を走らせる必要があるため、状態を保持しておく。

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
2. URL から source_kind（= 採用 Tier）を判定
   ├ youtube.com / youtu.be / shorts / live    → 'youtube_embed'  (Tier 1)
   ├ Content-Type が audio/* または .mp3 .m4a .ogg .aac → 'audio_file' (Tier 1, HEAD で判定)
   ├ tver.jp / その他「埋め込み拒否＋DRM」既知ホスト → 'extension_tab' (Tier 3)
   └ それ以外                                  → 'ytdlp_stream'   (Tier 2 候補)
3. items に INSERT (meta_status='pending', resolution_status='pending'|'na')
4. メタ取得を非同期で実行（プロセス内のキュー、せいぜい 4 並列）
   ├ youtube_embed  → oEmbed
   ├ audio_file     → HEAD で Content-Length, ファイル名から title 推定
   ├ ytdlp_stream   → yt-dlp -j で JSON 取得 → title/uploader/thumbnail を取り出す
   └ extension_tab  → og:* を正規表現で抽出（HTML を GET）
5. ytdlp_stream のときは続けて解決処理:
   ├ formats[] に DRM-only / 空 / 音声のみ → resolution_status='failed' に落として
   │   source_kind を 'extension_tab' に降格（Tier 3 へフォールバック）
   ├ HLS が取れる → media_url=manifest_url, stream_kind='hls', resolution_status='ok'
   └ progressive が取れる → media_url=直リン, stream_kind='progressive', resolution_status='ok'
6. items を UPDATE
```

`youtube_embed` も Tier 1 で再生失敗（`onError` 101/150）した場合に、
クライアント側から `POST /api/items/:id/refetch?fallback=ytdlp` を呼んで Tier 2 → Tier 3 に
降格させる経路を用意する。

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

### 6-3b. Tier 2 — yt-dlp 解決と HLS プロキシ

| Method | Path | 振る舞い |
|---|---|---|
| POST | `/api/items/:id/resolve` | yt-dlp を再実行してメタ＋ストリームを再解決（短期失効した m3u8 の再取得用） |
| GET  | `/api/stream/:itemId/manifest.m3u8` | items のマニフェストをバックエンドが取得して中継。`Access-Control-Allow-Origin: *` 付与、必要に応じてトークンも付与 |
| GET  | `/api/stream/:itemId/segment?u=<encoded-url>` | 上記マニフェスト内のセグメント URL を引数として受け取り、同じバックエンドが中継 |

`/api/stream/.../manifest.m3u8` の中身は **マニフェスト本体を書き換えて、セグメント URL を
`/api/stream/:itemId/segment?u=<encoded>` の形に差し替える**。これで CORS とトークン管理を
一箇所に閉じ込められる（[02-research.md](./02-research.md) §2-3）。

セキュリティ:

- `:itemId` で referencing されたアイテム以外の任意 URL は中継しない。
- セグメント URL は **マニフェスト解析時に DB 側で許可リストに追加**したホストのみ通す。
- これにより SSRF を構造的に防止する。

### 6-3c. Tier 3 — Chrome 拡張連携

| Method | Path | 振る舞い |
|---|---|---|
| POST | `/api/extension/play` | 本体 → 拡張へ「このタブ ID／URL で再生開始してくれ」を投げる（実態は `extension_state` に書いて、SSE で push） |
| GET  | `/api/extension/events` | SSE。拡張が `chrome.runtime.sendMessage` で本体に通知した `ended` / `error` / `progress` を、本体フロントへ流す |
| POST | `/api/extension/event` | 拡張のサービスワーカーが本体へ送る通知（HTTP POST。これを受けて本体は `extension_state` 更新と `/api/extension/events` への push を行う） |

> 拡張からは `externally_connectable` で本体（http://nagara.local:port）の HTTP API を直接叩いてもよい。
> ただし MV3 の service worker は短命なので、長時間の SSE 購読には向かない。
> **方向ごとに通信路を分ける**: 本体→拡張は `chrome.runtime.sendMessage`、
> 拡張→本体は `fetch`（短命 POST）。両方とも MV3 のサンプル実装で実証済みのパターン。

詳細フローは §7-2c。

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
│   [全画面]                  │   └────────────[追加]┘     │
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
唯一の抽象化レイヤー。MVP では実装 4 つ。**外部タブ手動進行のアダプタは存在しない**
（コンセプト破綻のため設計から除外）。

```ts
type PlayerEvent =
  | { type: "ended" }
  | { type: "error"; code: string; message: string }
  | { type: "fallback_required"; toTier: 2 | 3 }    // 上位 Tier から下位への降格依頼
  | { type: "needs_user_action"; reason: "autoplay_blocked" | "extension_missing" };

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

| 実装 | Tier | 中身 |
|---|---|---|
| `YouTubeEmbedAdapter` | 1 | `<div>` に YouTube IFrame Player API で `YT.Player` を生成。`onStateChange` の `ENDED` を `ended` に。`onError` 101/150 の場合は `fallback_required(toTier=2)` を発火 |
| `AudioFileAdapter` | 1 | `<audio>` を生成。`ended`／`error` を変換。`playbackRate` で速度。フルスクリーンは無効 |
| `HlsVideoAdapter` | 2 | `<video>` を生成し、`hls.js` で `/api/stream/:id/manifest.m3u8` をロード。`ended` / `error` をそのまま転送。フルスクリーンはコンテナで `requestFullscreen()` |
| `ExtensionTabAdapter` | 3 | 拡張インストール状態を `chrome.runtime.sendMessage(EXT_ID, {type:'ping'})` で確認。未インストールなら `needs_user_action(extension_missing)` を発火し再生を停止。インストール済みなら `POST /api/extension/play` を投げ、SSE 経由で `ended` を待ち受ける |

### 7-2c. Tier 3（Chrome 拡張モード）の往復シーケンス

```
[本体フロント]                                           [本体バックエンド]
     │                                                          │
     │ POST /api/extension/play  { itemId, sourceUrl }          │
     ├─────────────────────────────────────────────────────────►│
     │                                                          │
     │ ◄── chrome.runtime.sendMessage（SSE/直接 fetch どちらでも）
     │                                                          │
[Chrome 拡張 service worker]                                    │
     │                                                          │
     │ chrome.tabs.create({ url: sourceUrl, active: true })     │
     │                                                          │
[コンテンツスクリプト（対象ドメインで動く）]                     │
     │                                                          │
     │ document.querySelector('video') を監視                   │
     │ video.addEventListener('ended', ...)                     │
     │                                                          │
     │ ended 検知時:                                            │
     │ chrome.runtime.sendMessage  → service worker へ          │
     │                                                          │
     │ service worker: fetch('http://nagara.local/api/extension/event'
     │                       , { method:'POST', body:{ type:'ended', itemId } })
     ├─────────────────────────────────────────────────────────►│
     │                                                          │
     │ ◄─ SSE で本体フロントへ ended イベントを push ───────────│
     │                                                          │
     │ ExtensionTabAdapter.on(handler) が ended を発火          │
     │ → 上位レイヤがキュー次に進める                             │
```

**設計上の重要ポイント:**

- 拡張は **「ユーザ自身が普通にブラウザを使うのを自動化する」** だけ。DRM 解除・ストリーム保存はしない。
- `chrome.tabs.create` の戻り値の `tab.id` を保持しておき、次のエピソードに移るときは
  **同じタブの URL を `chrome.tabs.update(tabId, { url })` で差し替える**ことでタブの増殖を避ける。
  これによりブラウザ全体の負荷も抑えられ、autoplay の文脈も維持しやすい
  （[02-research.md](./02-research.md) §2-4 の autoplay 制約への対策）。
- 本体は拡張がインストールされていない／応答しないときに `extension_missing` イベントを上に流し、
  ユーザに **拡張インストールを促すバナー** を表示する。インストール後はこのアイテムから再開できる。

### 7-2d. Tier 2 の m3u8 短期失効への対処

`HlsVideoAdapter.load()` 時、`item.resolved_at` をサーバから受け取ってフロントで判定する:

```ts
const RESOLUTION_TTL = 30 * 60 * 1000; // 30 分（保守的に短め）
if (Date.now() - new Date(item.resolved_at).getTime() > RESOLUTION_TTL) {
  await fetch(`/api/items/${item.id}/resolve`, { method: 'POST' });
  // 再取得した item を使い直す
}
```

- TTL は設定可能。サイトごとの実態に応じて伸縮する余地を残す（`settings.ytdlp_resolution_ttl_seconds`）。
- 解決失敗（DRM 等）が後から発覚したら、サーバ側で `source_kind` を `extension_tab` に降格させる
  → 次回の `play()` 時に `ExtensionTabAdapter` が選ばれる。

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
nagara-app/            # ← 実装が始まったらここに置く想定（モノレポ形式）
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
│  │  ├ ytdlp/                 # Tier 2（yt-dlp 子プロセス、formats 解析）
│  │  │  ├ resolve.ts
│  │  │  └ classify.ts         # DRM/HLS/progressive 判定
│  │  ├ stream-proxy.ts        # Tier 2（HLS マニフェスト書換 + セグメント中継）
│  │  ├ extension-bridge.ts    # Tier 3（SSE エンドポイントと event 受け口）
│  │  └ routes/
│  │     ├ items.ts
│  │     ├ queue.ts
│  │     ├ history.ts
│  │     ├ settings.ts
│  │     ├ stream.ts
│  │     └ extension.ts
│  ├ web/
│  │  ├ index.html
│  │  ├ main.tsx
│  │  ├ player/
│  │  │  ├ PlayerAdapter.ts
│  │  │  ├ YouTubeEmbedAdapter.ts
│  │  │  ├ AudioFileAdapter.ts
│  │  │  ├ HlsVideoAdapter.ts          # Tier 2
│  │  │  └ ExtensionTabAdapter.ts      # Tier 3
│  │  ├ pages/
│  │  │  ├ Player.tsx
│  │  │  ├ History.tsx
│  │  │  └ Settings.tsx
│  │  └ components/...
│  └ extension/                # Chrome 拡張（MV3）
│     ├ manifest.json
│     ├ service-worker.ts
│     └ content/
│        └ video-watcher.ts
├ data/                # SQLite ファイル（gitignore）
└ scripts/
   ├ install-systemd.sh
   └ install-ytdlp.sh   # yt-dlp バイナリのインストール
```

## 9. 受け入れ条件（DoD）

依頼方針「ミニマムに作って触ってから設計を詰める」と
「ながら聞きコンセプトを破綻させない」を満たすため、初回リリースの DoD:

**Tier 1（ネイティブ埋め込み）**
- [ ] YouTube 動画 URL を貼ると、oEmbed でメタが埋まり、IFrame Player API で再生される。
- [ ] mp3 直リンを貼ると、メタが推定で埋まり、`<audio>` で再生される。
- [ ] エピソード末尾まで来たら自動で次に進む（人間操作なしで連続再生）。

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
- [ ] 次のエピソードが Tier 3 のときは、同じタブの URL を更新して使い回す（タブ増殖を防ぐ）。
- [ ] 拡張未インストール時は、Tier 3 アイテム再生開始時にインストール案内バナーが出る。

**コア体験（Tier 横断）**
- [ ] キューはドラッグで並び替えできる。削除も「次これ再生」も動く。
- [ ] ブラウザ再起動してもキューが復元される（SQLite 永続化）。
- [ ] 履歴画面で過去のアイテムを再投入できる。
- [ ] ザッピングタイマーは設定で有効化したときのみ動作する。

**運用**
- [ ] Linux PC で `node dist/server.js` を systemd に登録するだけで常駐できる。
- [ ] yt-dlp バイナリは `scripts/install-ytdlp.sh` で取得・更新できる。

ここまで満たせば「触ってみる」が成立する。
そこから運用してみて、自動キューイングや投稿者管理の必要性を見極めて
[04-roadmap.md](./04-roadmap.md) のロードマップに沿って積み増す。
