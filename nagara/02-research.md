# 02. 各サービスの取得・再生方式 調査メモ

設計を「想像」で組まないように、各配信サービスについて
**実際の仕様・既存実装** に当たった結果をここにまとめる。
本書は設計判断の根拠資料であって、フロー図のための雰囲気記述ではない。

## 1. YouTube

### 1-1. チャンネル新着の取得（API キー不要）

YouTube は公式に **RSS フィード** を提供している。

- URL 形式: `https://www.youtube.com/feeds/videos.xml?channel_id=<UC...>`
- API キー不要。フィードに含まれるのは直近 15 件程度（過去全件は取れない）。
- ハンドル形式 `@xxxx` の URL からは、ページソース内の `"channelId":"UC..."` を
  正規表現で拾うことでチャンネル ID を解決できる。
- これだけで「登録チャンネルの新着動画一覧」は十分得られる。

> 出典: <https://developers.google.com/youtube/iframe_api_reference> のサンプル、
> および公式 RSS の慣行（YouTube ヘルプセンタが該当 URL を案内）。

**MVP 方針:** YouTube Data API v3 を使わない。`feeds/videos.xml` のパースのみで足りる。
これにより API キー管理・Quota 管理が消えるので「ミニマム」の方針に合致する。

### 1-2. 埋め込み再生と JS 制御

公式の **YouTube IFrame Player API** で必要な操作はすべて満たせる。

確認した API 仕様（出典: [YouTube IFrame Player API Reference](https://developers.google.com/youtube/iframe_api_reference)）:

| 要求機能 | 提供されている API |
|---|---|
| ロード | `cueVideoById(videoId)` / `loadVideoById(videoId)` |
| 再生 | `playVideo()` |
| 一時停止 | `pauseVideo()` |
| 見終わり検知 | `onStateChange` イベントで `event.data === 0`（`YT.PlayerState.ENDED`）を捕捉 |
| 再生速度 | `setPlaybackRate(rate)` / `getAvailablePlaybackRates()` |
| 現在位置／長さ | `getCurrentTime()` / `getDuration()` |
| 埋め込み拒否動画の検知 | `onError` で `event.data === 101` または `150` |
| その他のエラー | `2`(無効パラメータ) / `5`(HTML5 不可) / `100`(動画なし) / `153` |

**注意点（実装で必須）:**

- **自動再生は音声付きでは原則ブロック** される（Chrome の Autoplay Policy）。
  - `<iframe>` の親要素から `allow="autoplay"` を付与した上で、
  - 最初の 1 本目は **ユーザーが明示的に「開始」ボタンを押したタイミングで `playVideo()` を呼ぶ**
    必要がある。
  - 2 本目以降は同一ドキュメント内で連鎖再生されるため、ユーザー操作の文脈が引き継がれる。
- iframe の `src` には `enablejsapi=1` と `origin=<こちらのオリジン>` を必ず付ける。

> 出典: MDN「Autoplay guide for media and Web Audio APIs」、
> Chrome Developers「Autoplay policy in Chrome」。

### 1-3. フルスクリーン

- iframe Player API には「フルスクリーン化」の専用メソッドは存在しない。
- フルスクリーン化はブラウザの **Fullscreen API**（`element.requestFullscreen()`）で行う。
  iframe をラップしている `<div>` に対して呼び出す。
- iframe 側のフルスクリーンボタン非表示は `playerVars.fs = 0` で抑制可能。

**「コメントスクリーンありフルスクリーン」について:**

要求にはあるが、コメント取得元が未定（YouTube ライブのチャットなのか、ニコニコ系なのか
独自コメントなのか）。MVP では実装せず、ロードマップに残す。

## 2. Podcast（汎用 RSS）

### 2-1. 取得仕様

- 標準的な RSS 2.0 ＋ iTunes Namespace。
- 各エピソード（`<item>`）から拾う情報:
  - `<title>` — タイトル
  - `<pubDate>` — 公開日時
  - `<guid>` — 一意 ID（`isPermaLink="false"` かどうかは無視してよい）
  - `<enclosure url=... type=... length=...>` — 音声ファイルの URL／MIME／サイズ
  - `<itunes:duration>` — 長さ。`HH:MM:SS` または秒のいずれか
- パーサは `rss-parser`（npm）が事実上の標準。iTunes namespace を組込みでサポート。

> 出典: Apple Podcasts for Creators「Podcast RSS feed requirements」、
> Podcast Standards Project の PSP-1 仕様。

### 2-2. 再生

- `<enclosure url>` を HTML5 `<audio src="...">` にそのまま流すだけで動く。
- CORS は再生（音声ストリーム取得）には影響しない（メディア要素は CORS 制約の対象外）。
  ただし「進捗を JS で読みたい」「波形を読みたい」場合のみ `crossorigin` 属性が要る。
  本ツールでは進捗（`audio.currentTime` / `audio.duration` / `ended` イベント）が取れれば
  十分なので、`crossorigin` 不要。

## 3. Radiko（MVP 対象外）

### 3-1. 現状

- **公式の埋め込み iframe／公開 API は提供されていない。**
- 既存実装（streamlink プラグイン、Kotlin での独自クライアント実装記事、
  Chrome 拡張 `rajiko` 等）はいずれも非公式の auth1/auth2 フローを叩いて
  HLS の `playlist.m3u8` を取得し、`X-Radiko-AuthToken` をヘッダに付けて再生する方式。
- 認証キー（`bcd1...`）の埋め込み、地域判定、プレミアム認証など実装コストが大きい。
- 利用規約への抵触懸念もあり、**プライベートツールであっても慎重に扱う領域**。

> 出典: streamlink/radiko.py、`jackyzy823/rajiko` README、
> 「2025年最新版 radiko APIでラジオアプリを作ろう」記事。

### 3-2. 結論

MVP では対象外。後から追加する場合は、**バックエンド側でのみ HLS を取得して
フロントへ中継する** 形（HLS プロキシ）にしないと、認証ヘッダ付きで配信し続けられない。

## 4. TVer（MVP 対象外）

### 4-1. 現状

- **公式の埋め込み iframe／公開 API は提供されていない。**
- Kodi 用アドオン `kuriho/plugin.video.tver` などのコミュニティ実装は内部で `yt-dlp` を呼んで
  m3u8 を抽出している。継続メンテが必要で、TVer 側の改修頻度が高い。
- DRM が掛かるコンテンツも多い。

### 4-2. 結論

MVP では対象外。導入するならローカルで `yt-dlp` を子プロセスで実行する経路が現実的だが、
本ツール本来の用途（「ながら聴き」）では音声情報量が乏しいので優先度は低い。

## 5. ブラウザ自動再生ポリシー（横断課題）

すべてのソースに共通する制約。

- **音声つきで自動再生したい場合、最初の一本目は必ずユーザー操作が要る。**
- 一度ユーザー操作が走れば、同一ドキュメント内では連続再生してよい。
- ページに `allow="autoplay"` を iframe に付ける。
- 失敗時のフォールバックとして「再生を開始する」ボタンを常に画面のどこかに置く。
  （`audio.play()` の Promise が `NotAllowedError` で reject されたら出す）

> 出典: MDN Autoplay guide、Chrome 開発者ブログ「Autoplay policy in Chrome」。

## 6. 任意 UA 指定（要望）

- フロントの `<iframe>` や `<audio>` 要素は **ブラウザの UA を上書きできない**。
  → ブラウザ側からの送信 UA を変える術は無い。
- できるのは **サーバー側で外部リソースを取得する際の UA 上書き** のみ。
  本ツールでは「YouTube の `feeds/videos.xml` 取得」「Podcast RSS 取得」がそれに当たる。
- MVP では環境変数 `OUTBOUND_USER_AGENT` を 1 個用意してサーバー fetch に渡すだけで足りる。
  汎用プロキシエンドポイントは MVP では作らない（SSRF 対策コストに見合わない）。
