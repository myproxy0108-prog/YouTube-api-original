
---

# YouTube Invidious-Compatible API (Cloudflare Workers)

Cloudflare Workers 上で動作する、Invidious 互換の超高速 YouTube API バックエンドエンジンです。
公式プレーヤーのエラー153対策、チャンネル全タブ（動画・ショート・再生リスト・ホーム）、動画の並べ替え、0ms連鎖プリフェッチが組み込まれています。

- **ベースURL**: `https://YOUR_WORKER_SUBDOMAIN.workers.dev`
- **通信形式**: JSON (`Content-Type: application/json`)
- **CORS**: 全オリジン許可 (`Access-Control-Allow-Origin: *`)
- **埋め込みドメイン**: `https://www.youtube-nocookie.com`

---

## エンドポイント一覧

| メソッド | パス | 説明 |
| :--- | :--- | :--- |
| `GET` | `/api/v1/search` | 動画・ショート・再生リスト検索（0ms追加読込） |
| `GET` | `/api/v1/channels/:id` | チャンネル情報（タブ切替・並べ替え・再生リスト） |
| `GET` | `/api/v1/shorts/:id` | ショート動画詳細 ＆ 次の動画3本（自動再生対応） |
| `GET` | `/api/v1/videos/:id` | 通常動画詳細 ＆ 関連動画 ＆ コメント一括ロード（目安3秒） |
| `GET` | `/api/v1/comments/:id` | コメント一覧取得（重複ゼロ・次ページ対応） |
| `GET` | `/api/v1/playlists/:id` | 再生リスト内の動画一覧取得 |
| `GET` | `/api/v1/trending` | 急上昇・トレンド動画一覧 |

---

## 1. 検索 (`/api/v1/search`)

通常動画・ショート動画・再生リストをまとめて検索します。
投機的プリフェッチ機能により、2ページ目以降の追加読み込みは **0ms〜数ms** で即返却されます。

### クエリパラメータ
- `q` (string): 検索キーワード（初回検索時は必須、追加読込時は不要）
- `continuation` (string): 続きを読み込むためのトークン
- `limit` (number): 取得件数（デフォルト: 30、最大: 60）

### リクエスト例
```bash
# 初回検索
GET /api/v1/search?q=Hikakin&limit=30

# 続きの読み込み
GET /api/v1/search?continuation=4qmFsg...
```

### レスポンス例
```json
{
  "results": [
    {
      "type": "video",
      "videoId": "SX_ViT4Ra7k",
      "title": "動画タイトル",
      "author": "チャンネル名",
      "authorId": "UCXuqSBlHAE6Xw-yeJA0Tunw",
      "thumbnail": "https://i.ytimg.com/vi/SX_ViT4Ra7k/hqdefault.jpg",
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
      "playlistId": "PLoSWVnSA...",
      "title": "再生リスト名",
      "videoCount": 12,
      "thumbnail": "https://i.ytimg.com/vi/.../hqdefault.jpg"
    }
  ],
  "continuation": "4qmFsg..."
}
```

---

## 2. チャンネル情報 (`/api/v1/channels/:id`)

チャンネルID（`UC...`）だけでなく、カスタムハンドル名（`@HikakinTV`）も自動解決します。
各タブの切り替えや、「新着順」「人気順」「古い順」の並べ替えに対応しています。

### パスパラメータ
- `:id` (string): チャンネルID (`UCXuqSBlHAE6Xw-yeJA0Tunw`) または ハンドル名 (`@HikakinTV`)

### クエリパラメータ
- `tab` (string): `videos`（動画、デフォルト）、`shorts`（ショート）、`playlists`（再生リスト）、`home`（ホーム）
- `sort` (string): `latest`（新着順、デフォルト）、`popular`（人気の動画）、`oldest`（古い順）
- `continuation` (string): チャンネル内動画の次ページトークン
- `limit` (number): 取得件数（デフォルト: 30）

### リクエスト例
```bash
# 人気順の動画一覧
GET /api/v1/channels/@HikakinTV?tab=videos&sort=popular

# ショート動画一覧
GET /api/v1/channels/@HikakinTV?tab=shorts

# 再生リスト一覧
GET /api/v1/channels/@HikakinTV?tab=playlists
```

### レスポンス例 (再生リストタブ時)
```json
{
  "author": "HikakinTV",
  "authorId": "UCXuqSBlHAE6Xw-yeJA0Tunw",
  "authorUrl": "/channel/UCXuqSBlHAE6Xw-yeJA0Tunw",
  "subCount": 19800000,
  "description": "チャンネル説明文...",
  "currentTab": "playlists",
  "currentSort": "latest",
  "tabs": ["home", "videos", "shorts", "playlists"],
  "contents": [
    {
      "type": "playlist",
      "title": "WBC2026 応援生配信",
      "playlistId": "PLoSWVnSA...",
      "videoCount": 3,
      "thumbnail": "https://i.ytimg.com/vi/.../hqdefault.jpg",
      "author": "HikakinTV"
    }
  ],
  "continuation": "..."
}
```

---

## 3. ショート動画 (`/api/v1/shorts/:id`)

ショート動画の情報と、**次に再生すべき厳選3本（シーケンス）**を返します。
自動再生とループ再生用のURL（`embedUrl`）が生成されます。

### パスパラメータ
- `:id` (string): ショート動画ID 11桁（省略時は初期ショート動画）

### リクエスト例
```bash
GET /api/v1/shorts/9f1rHt0W_Ak
```

### レスポンス例
```json
{
  "status": "success",
  "type": "shorts",
  "current": {
    "id": "9f1rHt0W_Ak",
    "title": "ショート動画タイトル",
    "author": "クリエイター名",
    "authorId": "UC...",
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
```

---

## 4. 通常動画詳細・関連動画 (`/api/v1/videos/:id`)

動画を開いた瞬間にすべての情報が揃うよう、**動画詳細・関連動画・初期コメント30件を最大3秒目安で同時一括ロード**して返します。

### パスパラメータ
- `:id` (string): 動画ID 11桁

### リクエスト例
```bash
GET /api/v1/videos/SX_ViT4Ra7k
```

### レスポンス例
```json
{
  "type": "video",
  "title": "動画タイトル",
  "videoId": "SX_ViT4Ra7k",
  "thumbnail": "https://i.ytimg.com/vi/SX_ViT4Ra7k/hqdefault.jpg",
  "description": "説明文...",
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
      "content": "コメント本文",
      "publishedText": "2日前",
      "likeCount": 120
    }
  ],
  "commentsContinuation": "..."
}
```

---

## 5. コメント単体取得 (`/api/v1/comments/:id`)

動画のコメント一覧を取得します。返信ボタンのトークン誤爆を防ぎ、同じコメントが再度読み込まれるバグを排除しています。

### クエリパラメータ
- `continuation` (string): コメントの次ページトークン
- `limit` (number): 取得件数（デフォルト: 50）

### リクエスト例
```bash
# 初回
GET /api/v1/comments/SX_ViT4Ra7k?limit=30

# 続きの読み込み
GET /api/v1/comments/SX_ViT4Ra7k?continuation=...
```

---

## 6. プレイリスト詳細 (`/api/v1/playlists/:id`)

指定した再生リストに含まれる動画一覧を取得します。

### パスパラメータ
- `:id` (string): プレイリストID（`PL...` または `VL...`）

### リクエスト例
```bash
GET /api/v1/playlists/PLoSWVnSA9vG8hI-SUpAimvYJrPh-PRRvp?limit=50
```

---

## 7. トレンド・急上昇 (`/api/v1/trending`)

急上昇動画の一覧を取得します。

### クエリパラメータ
- `type` (string): `default`（急上昇全体）、`music`（音楽）、`gaming`（ゲーム）

### リクエスト例
```bash
GET /api/v1/trending?type=music
```

---

## ⚠️ フロントエンド実装時の重要事項（エラー153対策）

YouTube埋め込みプレーヤーの仕様により、**HTTP Referer（参照元情報）が遮断されていると「動画プレイヤーの設定エラー（Error 153）」が発生**します。
正常に動画を再生させるため、HTML側に必ず以下の設定を行ってください。

### 1. HTMLのヘッダー設定
`no-referrer` は指定せず、YouTube公式推奨のポリシーを指定してください。
```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

### 2. iframeタグの設定
`referrerpolicy` 属性を付与して埋め込みます。
```html
<iframe 
  src="https://www.youtube-nocookie.com/embed/VIDEO_ID?autoplay=1" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
  allowfullscreen 
  referrerpolicy="strict-origin-when-cross-origin">
</iframe>
```
#注意点
1. 自作発言をしないでください
2. また、ここにベタ書きしているapiを使用しないでください
3. streamには対応していません、現存のinvidiousを少しでも軽くするために使用してください
4. 改造をする場合は、chatworkに来て、改造許可を僕にとってください-
