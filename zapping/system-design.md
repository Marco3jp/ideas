# システム設計

## アーキテクチャ概観

```
┌──────────────────────────────────────────────────────────┐
│                        ブラウザ                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           フロントエンド (SPA)                       │ │
│  │  ┌──────────────┐  ┌──────────────────────────────┐ │ │
│  │  │  Player UI   │  │  Queue / History / Settings  │ │ │
│  │  │  (iframe wrap│  │  管理画面                     │ │ │
│  │  │   per source)│  │                              │ │ │
│  │  └──────┬───────┘  └──────────────┬───────────────┘ │ │
│  └─────────┼────────────────────────┼─────────────────┘ │
│            │ postMessage / DOM event │ REST / WebSocket   │
└────────────┼────────────────────────┼───────────────────┘
             │                        │
             ▼                        ▼
┌─────────────────────────────────────────────────────────┐
│                    バックエンド (API サーバー)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │ Channel  │ │ Episode  │ │  Queue   │ │ History   │  │
│  │ Crawler  │ │ Filter   │ │ Builder  │ │ Recorder  │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Scheduler (cron / job queue)        │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
 YouTube       Podcast      （Radiko /
 Data API      RSS Feed       TVer 等）
```

---

## コンポーネント詳細

### フロントエンド

| コンポーネント | 役割 |
|---------------|------|
| Player UI | 各サービスの埋め込みプレイヤーを iframe でラップし、統一 API で操作 |
| Queue Viewer | 現在のキューを表示・手動操作（スキップ・削除・並び替え） |
| History Viewer | 視聴履歴の参照・除外設定 |
| Channel Manager | チャンネル・番組の登録・設定 |
| Filter Manager | 正規表現フィルタ・ミュートリストの管理 |

**技術スタック候補（MVP）**

- フレームワーク: Next.js (App Router) または Vite + React
- 状態管理: Zustand（軽量で十分）
- UI コンポーネント: shadcn/ui + Tailwind CSS
- プレイヤー通信: iframe postMessage ブリッジ

### バックエンド

| コンポーネント | 役割 |
|---------------|------|
| Channel Crawler | YouTube Data API / RSS フィードを定期取得し、新着エピソードを DB へ保存 |
| Episode Filter | フィルタルール・ミュートリストを適用し再生可否を判定 |
| Queue Builder | 視聴履歴・優先度ポリシーに基づき再生キューを生成 |
| History Recorder | 再生開始・完了・進捗率を記録 |
| Scheduler | 定期クロールと自動キュー更新をトリガー |

**技術スタック候補（MVP）**

- ランタイム: Node.js (TypeScript) / Hono または Fastify
- DB: SQLite（個人利用・シングルインスタンス）→ 将来は PostgreSQL
- ジョブキュー: BullMQ（Redis）または node-cron（MVP は cron で十分）
- YouTube: `googleapis` npm パッケージ
- RSS: `rss-parser` npm パッケージ

---

## データフロー

### 新着コンテンツ取得フロー

```
Scheduler (定期)
  → Channel Crawler
    → YouTube Data API / RSS
      → エピソード情報を DB へ upsert
        → Episode Filter で再生可否フラグを更新
          → Queue Builder がキューを再計算
```

### 再生フロー

```
フロントエンド起動
  → GET /api/queue → 現在のキューを取得
    → 先頭エピソードを Player UI にロード
      → iframe でサービス埋め込みプレイヤーを表示
        → 再生開始 → POST /api/history (played_at)
          → [ザッピングタイマー経過 or 見終わり検知]
            → POST /api/history (completed / progress)
              → GET /api/queue/next → 次エピソードへ
```

---

## プレイヤーブリッジ設計

各サービスのプレイヤーは埋め込み iframe であるため、直接 JS を操作できない。
以下の方式でコントロールを実現する。

| サービス | 方式 | 備考 |
|---------|------|------|
| YouTube | YouTube IFrame Player API（postMessage） | 公式 API あり |
| Podcast | `<audio>` タグを直接 DOM 操作 | RSS の `<enclosure>` URL を使用 |
| Radiko | Radiko Embed URL + postMessage（要調査） | 将来対応 |
| TVer | TVer Embed URL + postMessage（要調査） | 将来対応 |

**UA 変更対応**

- バックエンドのプロキシエンドポイントを経由してコンテンツをフェッチすることで、サーバーサイドで任意の UA を設定可能にする
- フロントエンドの `iframe` src は直接サービス URL を指さず、必要に応じてバックエンドプロキシ経由にする

---

## ザッピング機能の状態遷移

```
[再生中]
  ├─ (タイマー経過: 設定時間) ─→ [続き確認画面] (デフォルト10秒)
  │                                 ├─ ユーザーが「続きを見る」 → [再生中]
  │                                 └─ タイムアウト or「次へ」 → [次エピソードロード]
  └─ (見終わり検知)  ─────────────→ [次エピソードロード]

[次エピソードロード]
  → GET /api/queue/next
  → POST /api/history (completed)
  → 次エピソードを Player UI にロード
  → [再生中]
```

---

## セキュリティ考慮事項

- YouTube API Key 等はすべてサーバーサイドの環境変数で保持。フロントエンドに露出させない
- ツール自体はインターネット公開しないローカルネットワーク専用のため、外部からのアクセスリスクは最小
- MVP 段階では認証機能を省略する。ローカル LAN 内に閉じていることを前提とする
- 将来的に外部公開が必要になった場合は Basic 認証 / IP 制限を追加する

---

## デプロイ構成

ツール自体はローカルネットワーク内でのみ動作する。インターネットへの公開は不要。

### パターン A: Linux PC をサーバーにして Windows PC から LAN 接続

```
[Linux PC]
  docker compose up
    ├── frontend  :3000   ← LAN 内 Windows PC ブラウザが http://<linuxpc-ip>:3000 でアクセス
    ├── backend   :3001
    └── db (SQLite file mount)

[Windows PC ブラウザ]
  http://192.168.x.x:3000  （LAN 内 IP）
```

### パターン B: Windows PC 単体で起動してローカルで使う

```
[Windows PC]
  docker compose up（Docker Desktop）または Node.js 直接起動
    ├── frontend  :3000
    ├── backend   :3001
    └── db (SQLite file)

  ブラウザで http://localhost:3000 にアクセス
```

### 共通事項

- Docker Compose を使うことで Linux / Windows どちらでも同一の手順で起動可能
- SQLite をファイルマウントするためデータはホスト側に永続化される
- YouTube 動画・RSS 等の外部コンテンツはサーバーサイドがインターネットへ取得しに行く

---

## 技術的リスクと対策

| リスク | 内容 | 対策 |
|--------|------|------|
| YouTube IFrame API の制限 | 埋め込み禁止動画や API 変更 | 対象外動画はキューから自動除外、フォールバック処理 |
| Radiko / TVer のスクレイピング | 利用規約・構造変更リスク | MVP では対象外、将来対応時に公式 API / 拡張機能方式を検討 |
| RSS フィードの形式差異 | フィードごとに `<enclosure>` や duration が異なる | パーサーを抽象化し、サービスごとにアダプターを実装 |
| CORS | 各サービスのコンテンツを直接フェッチできない | バックエンドプロキシで解決 |
