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

### 1-4. TVer — 埋め込み拒否を確認

TVer の番組ページは **iframe 埋め込みが拒否される**。

- HTTP レスポンスヘッダで `X-Frame-Options` または CSP の `frame-ancestors` により
  自サイト以外の iframe からの読み込みをブロックしている。
- TVer 利用規約でも「サービスの構成要素のブロック・非表示・妨害」が禁止されている。

> 出典: 関連ブラウザ挙動ドキュメント（CSP frame-ancestors の解説記事 iret.media、Qiita）、
> TVer 利用規約 <https://tver.jp/tos>。

**設計上の結論:**
本ツール内で TVer を **iframe 再生することは技術的にも規約上もしない**。
代わりに「キューに混ぜる」 + 「再生時は外部タブで開く + ユーザが手動で『見終わった→次へ』を押す」
という運用にする。これによりプラットフォーム横断キューに TVer も混ぜられる。

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
再生は外部タブ任せになるが「手動キュー」の体験は維持できる。

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
  「外部タブで開く」モードに自動フォールバックして次へ進める。
  → ユーザは何も意識せず再生を続けられる。

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

### 2-3. 「埋め込み不可」ソース — 外部タブ + 手動進行

TVer のように iframe 拒否されるソース、および本ツールが対応していないプラットフォームについては、
**新規タブで対象 URL を開くだけ**。再生制御はそのプラットフォーム側に任せる。

UI 側は次の状態を持つ:
- 「外部で視聴中」表示
- 「視聴完了 → 次へ」ボタン
- 「視聴前に戻す」ボタン
- ザッピングタイマー（手動）

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
