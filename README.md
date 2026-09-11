GitHub等の README.md にそのままコピペして使える、API仕様書（APIリファレンス）の完全版です。

## 🚀 API リファレンス (API Documentation)

本プロジェクトは Cloudflare Workers 上で動作する超高速・軽量な YouTube API バックエンドエンジンです。Invidious 互換のエンドポイントを備え、キャッシュ・自動再生・チャンネル各タブ・エラー153完全対策が組み込まれています。

- **Base URL**: `https://<あなたのWorkerドメイン>.workers.dev`
- **通信形式**: JSON (`Content-Type: application/json; charset=utf-8`)
- **CORS**: 全オリジン対応 (`Access-Control-Allow-Origin: *`)
- **埋め込みドメイン**: 全て `https://www.youtube-nocookie.com` に統一

---

### 📌 エンドポイント一覧

| メソッド | エンドポイント | 説明 |
| :--- | :--- | :--- |
| `GET` | `/api/v1/search` | キーワード検索（動画・ショート・再生リスト・0ms追加読込） |
| `GET` | `/api/v1/channels/:id` | チャンネル情報（動画/ショート/再生リスト/ホーム ＆ 並べ替え） |
| `GET` | `/api/v1/shorts/:id` | ショート動画詳細 ＆ 次の厳選3本シーケンス |
| `GET` | `/api/v1/videos/:id` | 通常動画詳細 ＆ 関連動画 ＆ コメント一括ロード（目安3秒） |
| `GET` | `/api/v1/comments/:id` | コメント一覧取得（重複ゼロ・次ページ対応） |
| `GET` | `/api/v1/playlists/:id` | プレイリスト内の動画一覧取得 |
| `GET` | `/api/v1/trending` | 急上昇・トレンド動画取得 |

---

### 1. 検索 (`/api/v1/search`)
通常動画・ショート動画・プレイリストを網羅して検索します。
投機的プリフェッチにより、2ページ目以降の追加読み込みは **0ms〜数ms（爆速）** で返却されます。

#### クエリパラメータ
| パラメータ | 型 | 必須 | デフォルト | 説明 |
| :--- | :--- | :--- | :--- | :--- |
| `q` | string | ○* | - | 検索キーワード (*追加読込時は不要) |
| `continuation` | string | - | - | 続きを読み込むためのページネーショントークン |
| `limit` | number | - | `30` | 取得件数 (最大 60) |

#### レスポンス例
```json
{
  "results": [
    {
      "type": "video",
      "videoId": "SX_ViT4Ra7k",
      "title": "動画タイトル",
      "author": "チャンネル名",
      "authorId": "UC...",
      "authorUrl": "/channel/UC...",
      "thumbnail": "https://i.ytimg.com/vi/SX_ViT4Ra7k/hqdefault.jpg",
      "videoThumbnails": [ ... ],
      "lengthSeconds": 245,
      "viewCount": 150000,
      "viewCountText": "15万 回視聴",
      "publishedText": "3日前",
      "embedUrl": "https://www.youtube-nocookie.com/embed/SX_ViT4Ra7k?autoplay=1"
    },
    {
      "type": "short",
      "isShort": true,
      "videoId": "9f1rHt0W_Ak",
      "title": "ショート動画タイトル #shorts",
      "author": "クリエイター名",
      "embedUrl": "https://www.youtube-nocookie.com/embed/9f1rHt0W_Ak?autoplay=1&mute=1&controls=0&loop=1&playlist=9f1rHt0W_Ak&playsinline=1&enablejsapi=1&rel=0"
    },
    {
      "type": "playlist",
      "playlistId": "PL...",
      "title": "再生リスト名",
      "videoCount": 24,
      "thumbnail": "https://i.ytimg.com/vi/.../hqdefault.jpg"
    }
  ],
  "continuation": "4qmFsg..."
}

2. チャンネル情報・コンテンツ (/api/v1/channels/:id)

UC... 形式のチャンネルIDはもちろん、@HikakinTV のようなカスタムハンドル名も0msキャッシュ付きで自動解決します。
タブ選択および「新着順」「人気の動画」「古い順」の並べ替えに対応しています。

パスパラメータ

  - :id: チャンネルID (UCXuqSBlHAE6Xw-yeJA0Tunw) または ハンドル名 (@HikakinTV)

クエリパラメータ

| パラメータ          | 型      | デフォルト    | 選択肢・説明                                                                  |
| :------------- | :----- | :------- | :---------------------------------------------------------------------- |
| `tab`          | string | `videos` | タブ指定: `videos` (動画), `shorts` (ショート), `playlists` (再生リスト), `home` (ホーム) |
| `sort`         | string | `latest` | 並べ替え: `latest` (新着順), `popular` (人気の動画), `oldest` (古い順) ※動画・ショートタブ時有効   |
| `continuation` | string | \-       | タブ内の続きを読み込むトークン                                                         |
| `limit`        | number | `30`     | 取得件数 (最大 60)                                                            |

レスポンス例 (tab=playlists 時)

{
  "author": "HikakinTV",
  "authorId": "UCXuqSBlHAE6Xw-yeJA0Tunw",
  "authorUrl": "/channel/UCXuqSBlHAE6Xw-yeJA0Tunw",
  "authorThumbnails": [ ... ],
  "authorBanners": [ ... ],
  "subCount": 19800000,
  "description": "チャンネル説明文...",
  "currentTab": "playlists",
  "currentSort": "latest",
  "tabs": ["home", "videos", "shorts", "playlists"],
  "contents": [
    {
      "type": "playlist",
      "title": "WBC2026 応援生配信",
      "playlistId": "PL...",
      "videoCount": 3,
      "thumbnail": "https://i.ytimg.com/vi/.../hqdefault.jpg",
      "author": "HikakinTV"
    }
  ],
  "continuation": "..."
}

3. ショート動画 (/api/v1/shorts/:id)

ショート動画単体情報と、**次に再生すべき厳選3本（シーケンス）**を同時に返却します。 自動ループ再生用のnocookie
URLがあらかじめ生成されています。

パスパラメータ

  - :id: ショート動画ID (9f1rHt0W_Ak) ※未指定時はおすすめの初期ショートを返却

レスポンス例

{
  "status": "success",
  "type": "shorts",
  "current": {
    "id": "9f1rHt0W_Ak",
    "title": "ショート動画タイトル",
    "author": "クリエイター名",
    "authorId": "UC...",
    "authorThumbnails": [ ... ],
    "likeCount": "12万",
    "commentCount": "コメント",
    "embedUrl": "https://www.youtube-nocookie.com/embed/9f1rHt0W_Ak?autoplay=1&mute=1&controls=0&loop=1&playlist=9f1rHt0W_Ak&playsinline=1&enablejsapi=1&rel=0",
    "videoThumbnails": [ ... ]
  },
  "sequence": [
    {
      "id": "abc12345678",
      "title": "次のショート1",
      "author": "投稿者名",
      "thumbnail": "https://i.ytimg.com/vi/abc12345678/hqdefault.jpg",
      "embedUrl": "https://www.youtube-nocookie.com/embed/abc12345678?..."
    }
  ]
}

4. 通常動画詳細 ＆ コメント同梱ロード (/api/v1/videos/:id)

動画を開いた瞬間に快適に視聴できるよう、動画情報・おすすめ動画・初期コメント（30件）を最大3秒目安で並行一括ロードして返します。フロント側でコメント用の別リクエストを待つ必要がありません。

パスパラメータ

  - :id: 動画ID 11桁 (SX_ViT4Ra7k)

レスポンス例

{
  "type": "video",
  "title": "動画タイトル",
  "videoId": "SX_ViT4Ra7k",
  "thumbnail": "https://i.ytimg.com/vi/SX_ViT4Ra7k/hqdefault.jpg",
  "description": "動画の説明欄テキスト...",
  "publishedText": "2026/01/15",
  "viewCount": 2400000,
  "author": "アーティスト名",
  "authorId": "UC...",
  "subCountText": "チャンネル登録者数 500万人",
  "embedUrl": "https://www.youtube-nocookie.com/embed/SX_ViT4Ra7k?autoplay=1",
  "recommendedVideos": [ ... ],
  "commentCount": 3500,
  "comments": [
    {
      "author": "ユーザー名",
      "authorThumbnails": [ ... ],
      "content": "コメント本文",
      "publishedText": "2日前",
      "likeCount": 120
    }
  ],
  "commentsContinuation": "コメント2ページ目用トークン"
}

5. コメント単体取得 (/api/v1/comments/:id)

動画のコメント一覧を取得します。返信ボタンのトークン誤爆を防ぎ、重複のない安定したページネーションを実現しています。

パスパラメータ / クエリパラメータ

  - :id: 動画ID 11桁
  - continuation (string, optional): コメントの次ページトークン
  - limit (number, default: 50): 取得件数

6. プレイリスト詳細 (/api/v1/playlists/:id)

指定した再生リストに含まれる動画一覧を取得します。

パスパラメータ

  - :id: プレイリストID (PL... または VL...)
  - limit (number, default: 50): 取得上限数

7. 急上昇・トレンド (/api/v1/trending)

日本国内の急上昇動画を取得します。

クエリパラメータ

  - type: default (急上昇全体), music (音楽), gaming (ゲーム)

⚠️ フロントエンド実装時の注意点（エラー153対策）

YouTubeの埋め込みプレイヤーが 動画プレイヤーの設定エラー (Error 153) を吐く現象を防ぐため、HTML側で必ず以下の設定を行ってください。

1.  HTML <head> 内の Referrer 設定:

    <!-- no-referrer だとエラー153が発生するため、公式推奨値を指定 -->
    <meta name="referrer" content="strict-origin-when-cross-origin">

2.  <iframe> の属性設定:

    <iframe 
      src="https://www.youtube-nocookie.com/embed/VIDEO_ID?autoplay=1" 
      frameborder="0" 
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
      allowfullscreen 
      referrerpolicy="strict-origin-when-cross-origin">
    </iframe>
#注意点
1. 自作発言をしないでください
2. また、ここにベタ書きしているapiを使用しないでください
3. streamには対応していません、現存のinvidiousを少しでも軽くするために使用してください
4. 改造をする場合は、chatworkに来て、改造許可を僕にとってください-
