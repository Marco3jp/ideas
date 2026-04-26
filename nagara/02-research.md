# 02. 各サービスの取得・再生方式 調査メモ

設計を「想像」で組まないように、各配信サービスについて
**実際の仕様・既存実装** に当たった結果をここにまとめる。
本書は設計判断の根拠資料であって、フロー図のための雰囲気記述ではない。

## 1. URL 単体からメタデータを取り出す（手動キュー前提のコア）

手動キューでは「URL 1 本貼ったらタイトル・投稿者・サムネが入る」が必要になる。
プラットフォームごとの取得手段を確認する。

### 1-1. YouTube — oEmbed エンドポイント（API キー不要）

公式の oEmbed エンドポイントが API キーなしで使える。

- URL: `https://www.youtube.com/oembed?url=<video-url>&format=json`
- HTTP 200 の場合、JSON で以下のフィールドが返る:
  - `title`
  - `author_name`（チャンネル名）
  - `author_url`（チャンネル URL）
  - `thumbnail_url`／`thumbnail_width`／`thumbnail_height`
  - `html`（埋め込み用 HTML：そのまま貼ればよいが、本ツールは IFrame Player API を直接使うので不要）
  - `provider_name` / `provider_url` / `type` / `version`
- HTTP 401 が返る → 動画が非公開／埋め込み禁止。
- HTTP 404 が返る → 動画が存在しない／削除済み。

> 出典: 公式 oEmbed エンドポイントは Stack Overflow の検証回答、
> および各種解説（uber-rob.co.uk、abdus.dev、queen.raae.codes 等）でキー不要で利用可能と確認できる。
> Wagtail CMS の解説では `rel=0` 等の埋め込みパラメータが返却 HTML に保持されない既知の挙動も指摘されている
> （本ツールは `html` を使わないので影響なし）。

**実装方針:**
URL 入力 → URL から videoId を抽出 → oEmbed を fetch → メタ確定。
videoId 抽出のための URL パターン:
- `https://www.youtube.com/watch?v=<id>`
- `https://youtu.be/<id>`
- `https://www.youtube.com/shorts/<id>`
- `https://www.youtube.com/live/<id>`

**注意点:**
oEmbed には **再生時間（duration）が含まれない**。
duration が必要なら YouTube Data API v3 が必要だが、本 MVP では duration は無くてもキューが成立する
（プログレスバーは IFrame Player API の `getDuration()` で取得できるため、保存不要）。

### 1-2. YouTube — チャンネル新着 RSS（参考；MVP 対象外）

`https://www.youtube.com/feeds/videos.xml?channel_id=UC...` で API キー不要に新着が取れる。
ただし本 MVP ではチャンネル登録機能を作らないので、**MVP では使わない**。
ロードマップで自動キューイング機能を作るときに採用する候補（[04-roadmap.md](./04-roadmap.md)）。

### 1-3. Podcast の RSS

- 標準的な RSS 2.0 ＋ iTunes Namespace。
- 各 `<item>` から拾うのは `<title>`／`<pubDate>`／`<guid>`／
  `<enclosure url type length>`／`<itunes:duration>`。
- パーサは `fast-xml-parser`（npm）。

> 出典: Apple Podcasts for Creators「Podcast RSS feed requirements」、
> Podcast Standards Project の PSP-1 仕様。

**MVP での使い方（手動キュー文脈）:**
- ユーザが Podcast 番組の RSS フィード URL を貼った場合 → 一覧を返してユーザに選ばせる。
- ユーザが Podcast の単一エピソード（mp3 直 URL や、Apple/Spotify のエピソード URL）を貼った場合
  - 直リンの mp3/m4a → そのまま `<audio>` で再生（メタは Content-Type と `Content-Length`、
    ファイル名から推定）。
  - Apple Podcasts や Spotify のエピソードページ URL → og:タグスクレイピングで title／author を拾う
    （後述 §1-5）。

### 1-4. TVer — 埋め込み拒否と DRM 採用を確認

TVer の番組ページは **iframe 埋め込みが拒否され、かつ Widevine DRM が掛かっている**。

- HTTP レスポンスヘッダで `X-Frame-Options` または CSP の `frame-ancestors` により
  自サイト以外の iframe からの読み込みをブロックしている。
- TVer 利用規約でも「サービスの構成要素のブロック・非表示・妨害」が禁止されている。
- 配信基盤は Brightcove で、**Google Widevine による DRM 保護** が適用されている。
  yt-dlp の TVer エクストラクタは存在するが、DRM 対象セグメントは
  「DRM protected」としてスキップされる（[yt-dlp #13477](https://github.com/yt-dlp/yt-dlp/issues/13477)）。
  「`x-streaks-api-key` 付きで一時的に通る」「日本国内 IP でも 403 が出る」など
  仕様変更も頻繁（[yt-dlp #13874](https://github.com/yt-dlp/yt-dlp/issues/13874) /
  [#13888](https://github.com/yt-dlp/yt-dlp/issues/13888)）。

> 出典: 関連ブラウザ挙動ドキュメント（CSP frame-ancestors の解説記事 iret.media、Qiita）、
> TVer 利用規約 <https://tver.jp/tos>、yt-dlp の関連 Issue 群、
> 「TVer Videos in 2026」解説記事（DRM 採用と short token expiration の指摘）。

**設計上の結論:**

- iframe 埋め込みは技術的・規約的に不可。
- yt-dlp で抽出した m3u8 をアプリ内 `hls.js` で再生する経路（後述 Tier 2）も
  **DRM のため成立しない**（音声のみ・低画質に落としても結局再生キーが必要）。
- したがって TVer をながら聞きキューに混ぜるためには、
  **「ブラウザのタブ自体を再生面として使い、それを Chrome 拡張で自動制御する」（Tier 3）** しかない。
  公式プレイヤーを TVer 側のドメインで動かす形になり、DRM 制約も TVer 利用規約も
  ユーザのブラウザ通常利用と同じ枠で満たせる。

メタ取得は番組ページの og:タグ（`og:title` / `og:image` / `og:description`）をサーバー側で
スクレイピングして拾う（§1-5 と同じ汎用ロジック）。

### 1-5. 汎用 og:タグスクレイピング（フォールバック）

URL から oEmbed や RSS 経路でメタが取れなかったとき、
ページ HTML の `<meta property="og:*">` を読むだけで多くのプラットフォームに対応できる。

- 取れる情報:
  - `og:title`
  - `og:description`
  - `og:image`
  - `og:site_name`
- 取得は単純な HTTP GET + HTML 内 meta タグの抽出のみ（DOM パーサ不要、正規表現でも足りる）。
- `User-Agent` を指定可能にする（サイトによっては Bot を除外する）。

これがあるおかげで、**未対応プラットフォームでも URL を貼ってキューに積むこと自体は可能**。
ただし「ながら聞き」を成立させるためには、再生経路として §2-1〜§2-4 のいずれか（Tier 1〜3）に
落ちる必要がある。og:タグ単独はメタ取得用のフォールバックでしかない。

## 2. 再生（プラットフォーム別）

### 2-1. YouTube IFrame Player API（埋め込み再生）

公式の YouTube IFrame Player API で本ツールに必要な操作はすべて満たせる。

| 要求 | API |
|---|---|
| ロード | `cueVideoById(videoId)` / `loadVideoById(videoId)` |
| 再生 | `playVideo()` |
| 一時停止 | `pauseVideo()` |
| 見終わり検知 | `onStateChange` で `event.data === 0`（`YT.PlayerState.ENDED`） |
| 再生速度 | `setPlaybackRate(rate)` / `getAvailablePlaybackRates()` |
| 現在位置／長さ | `getCurrentTime()` / `getDuration()` |
| 埋め込み拒否動画の検知 | `onError` で `event.data === 101` または `150` |

> 出典: 公式 [YouTube IFrame Player API Reference](https://developers.google.com/youtube/iframe_api_reference)。

**注意点（実装で必須）:**
- 音声つき自動再生は原則ブロックされる（Chrome Autoplay Policy）。
  - `<iframe>` 親に `allow="autoplay"` を付与した上で、
  - 最初の 1 本目は **ユーザーが「再生開始」ボタンを押した瞬間** に `playVideo()` を呼ぶ。
  - 同一ドキュメント内であれば、その後の連鎖再生はユーザー操作の文脈が引き継がれる。
- iframe `src` には `enablejsapi=1` と `origin=<こちらのオリジン>` を付ける。
- 埋め込み拒否動画 (onError 101/150) を検知したら、
  **Tier 2（yt-dlp + hls.js）に自動フォールバック**する。Tier 2 でも DRM 等で失敗するなら
  Tier 3（拡張モード）に降格する。いずれの場合もユーザは何も意識せず再生を続けられる。

> 出典: MDN「Autoplay guide for media and Web Audio APIs」、
> Chrome Developers「Autoplay policy in Chrome」。

### 2-2. Podcast / 音声直リン — `<audio>` 要素

- `<audio src="...">` で素直に再生できる。
- 必要なイベント・プロパティはすべて HTMLMediaElement の標準:
  - `play()` / `pause()`
  - `playbackRate`
  - `currentTime` / `duration`
  - `ended` イベント
- CORS は再生（音声ストリーム取得）には影響しない（メディア要素は CORS 制約の対象外）。

### 2-3. yt-dlp 抽出 → アプリ内 `hls.js` 再生（Tier 2）

iframe 埋め込みが効かないが DRM が無いプラットフォームを「ながら聞き」可能にする経路。

- **対応サイト規模:** yt-dlp は **1000+ サイト** に対応。各サイトの URL を渡すと
  抽出可能なメディアの formats を JSON で返す。
- **JSON 構造:** `yt-dlp -j <url>` で 1 行の JSON を出力。`formats[]` に
  以下のフィールドが入る:
  - `protocol`: `m3u8` / `m3u8_native` / `https` / `http` 等。**`m3u8` 系なら HLS、`https` なら progressive MP4** で見分けられる。
  - `url`: メディア（progressive のときは直リン、HLS のときはマスタープレイリスト URL）。
  - `manifest_url`: HLS / DASH のマニフェスト URL（存在すれば streaming 形式の証拠）。
  - `vcodec` / `acodec` / `width` / `height` / `ext`。
- **DRM の判定:** yt-dlp は DRM 検出すると当該フォーマットをスキップし、
  ログに「DRM protected」と出す。**JSON の `formats[]` が空、または音声のみ／極端に低画質しか
  返ってこないものは DRM と判定** して Tier 2 を諦める。

> 出典: yt-dlp ドキュメント（`-j` の OUTPUT TEMPLATE 仕様）、
> [yt-dlp/yt-dlp#6213](https://github.com/yt-dlp/yt-dlp/issues/6213)、
> [yt-dlp/yt-dlp#13477](https://github.com/yt-dlp/yt-dlp/issues/13477)、
> `pkg.go.dev/github.com/lrstanley/go-ytdlp` の formats 解説。

**フロント側の再生:**

- HLS は `hls.js` を使う（Chrome は HLS をネイティブで `<video>` に流せないため）。
- progressive MP4 は `<video src="...">` だけで再生可能。
- いずれの場合も `<video>` 要素なので、`ended` / `play` / `pause` / `playbackRate` /
  `requestFullscreen()` がそのまま使える → Tier 1 と同じ操作感を保てる。

**hls.js の認証ヘッダ／CORS 対策:**

- m3u8／セグメントが配信元のままだと CORS でブロックされる場合がある。
- バックエンド側でセグメントもプロキシする方式が確実
  （`Access-Control-Allow-Origin: *` を強制付与、必要ならトークンも引き継ぐ）。
- hls.js は `xhrSetup` フックで XHR にヘッダ付与可能（マニフェストにもセグメントにも適用される）。
  ただし MVP では「**バックエンドプロキシ経由に統一**」する方が単純で、
  CORS／トークン期限／クッキーの面倒を一括して避けられる。

> 出典: hls.js のドキュメントおよび関連 Issue
> ([video-dev/hls.js#2331](https://github.com/video-dev/hls.js/issues/2331)、
> [#1068](https://github.com/video-dev/hls.js/issues/1068))、
> Stack Overflow「HLS.js required send http header」、関連解説記事。

### 2-4. Chrome 拡張モード（Tier 3）

Tier 1／Tier 2 で再生不能なプラットフォーム（典型例: TVer）を「ながら聞き」可能にするための経路。
**追加フィードバックで明示的に提案された案**。

#### 仕組み

```
nagara web (http://nagara.local:port)
  ↑                                  ↑
  │  externally_connectable          │  ws / SSE で「ended/error」通知
  │  経由で onMessageExternal        │
  ▼                                  │
Chrome 拡張（MV3, service worker）   │
  │                                  │
  │  chrome.tabs.create({ url })     │
  ▼                                  │
別タブ（YouTube / TVer / etc.）      │
  └─ コンテンツスクリプトが video 要素監視 ──┘
      （ended / error / pause / progress を本体へ転送）
```

#### 確認した API 仕様

- **`externally_connectable`:** manifest.json の `externally_connectable.matches` に
  `http://nagara.local/*` 等を列挙すると、その URL から `chrome.runtime.sendMessage(EXTENSION_ID, msg)`
  で拡張へメッセージが投げられる。拡張側は service worker の `chrome.runtime.onMessageExternal`
  で受信。MV3 で正式サポートされている。
- **`chrome.tabs.create({ url })` / `chrome.tabs.remove(tabId)`:** タブの起動と破棄が拡張権限で可能。
- **コンテンツスクリプト:** タブ内の `document.querySelector('video')` を取得し、
  `addEventListener('ended', ...)` で見終わりを検知して service worker へ `chrome.runtime.sendMessage`、
  そこから本体へ転送する。`playbackRate` の設定もコンテンツスクリプトから DOM 経由で可能。
- **本体 → 拡張の方向は service worker から:** コンテンツスクリプトは
  `chrome.tabs.sendMessage(tabId, msg)` で操作可能。

> 出典: Chrome for Developers「externally_connectable」公式ドキュメント、
> Stack Overflow「Chrome manifest V3 extensions and externally_connectable」、
> Qiita「Chrome拡張機能開発で役立つTips集（Manifest V3対応）」。

#### 既知の制約と対策

| 制約 | 対策 |
|---|---|
| 別タブの autoplay は元タブ以上に厳しく、最初の 1 回はユーザー操作が要る | 「再生開始」ボタン押下のジェスチャを起点に `chrome.tabs.create` を呼ぶ。連鎖再生時は既存タブを再利用して URL を差し替える（DOM 上は同一ドキュメント） |
| バックグラウンドタブで音声付き動画が自動 pause されるサイトがある（YouTube/Twitch 等の挙動報告あり） | nagara 用タブを **アクティブにしておく** 運用が前提。プレイヤー画面はキュー操作だけで、再生表示は拡張タブ側に任せる構成にする |
| DRM サイト（TVer 等）の規約 | 拡張は「ユーザー自身のブラウザでの通常閲覧を自動化しているだけ」であり、DRM 解除はしない。ストリームの保存もしない。表示は当該サイトの公式プレイヤーが行う |
| 拡張の配布 | 開発者モードでローカル `chrome://extensions/` から読み込む形を MVP とする。Chrome Web Store 公開は将来 |

#### 自動再生の制御（横断課題と同じ）

- `<iframe>` 親要素には `allow="autoplay"` を付ける。Chrome 拡張のタブそのものは
  Chrome のオートプレイポリシーがそのまま適用される。
- 「セッション開始時の最初の 1 アクション」がユーザーから出ていれば、
  service worker は連鎖して `chrome.tabs.update` で次の URL に飛ばせる。
- ユーザーが Chrome ウィンドウからフォーカスを外しても、
  service worker のイベントは動き続ける（タブが破棄されない限り）。

## 3. ブラウザ自動再生ポリシー（横断課題）

- 音声つきで自動再生したい場合、最初の一本目は必ずユーザー操作が要る。
- 一度ユーザー操作が走れば、同一ドキュメント内では連続再生してよい。
- iframe には `allow="autoplay"` を付ける。
- フォールバックとして「再生開始」ボタンを画面のどこかに常設し、
  `play()` の Promise が `NotAllowedError` で reject されたら明示する。

> 出典: MDN Autoplay guide、Chrome 開発者ブログ「Autoplay policy in Chrome」。

## 4. 任意 UA 指定（要望）

- フロントの `<iframe>` や `<audio>` 要素はブラウザの UA を上書きできない。
  → ブラウザ側からの送信 UA を変える術は無い。
- できるのは **サーバー側で外部リソースを取得する際の UA 上書き**。
  本ツールではメタ取得 fetch（oEmbed／og:タグスクレイピング／Podcast RSS）が対象。
- MVP では環境変数 `OUTBOUND_USER_AGENT` を 1 個用意してサーバー fetch に渡すだけで足りる。
  汎用プロキシエンドポイントは MVP では作らない（SSRF 対策コストに見合わない）。

## 5. Radiko（MVP 対象外）

公式の埋め込み API・iframe は提供されておらず、独自に auth1/auth2 フローを叩いて
HLS（`X-Radiko-AuthToken` 必須）を取得する必要があり実装コストが高い。

> 出典: streamlink/radiko.py、`jackyzy823/rajiko` README、
> 「2025年最新版 radiko APIでラジオアプリを作ろう」記事。

導入する場合はバックエンドで auth フローを完結させ、
HLS をフロントへプロキシ配信する構成になる。優先度は低（[04-roadmap.md](./04-roadmap.md)）。
