# ⚡ YouTube Invidious-Compatible Turbo API (Cloudflare Workers)by Nemu

YouTube公式APIキー（利用枠・課金制限）を一切使用せず、Cloudflare Workers 上で動作する Invidious 互換の高速 RESTful API エンジンです。  
動画詳細、関連動画、コメント、チャンネル、プレイリスト、急上昇（トレンド）、およびショート動画（スワイプ次動画取得対応）を JSON 形式で提供します。

---

## 🌟 特徴

- **API キー完全不要**: 制限や割当枠（Quota）を気にせず利用可能。
- **Invidious スキーマ完全準拠**: 既存の Invidious クライアントアプリや自作フロントエンドにそのまま接続可能。
- **高速応答**: エッジメモリキャッシュ（5分間）を内蔵し、同一リクエストには 0ms レベルで即返却。
- **全エンドポイント CORS 対応**: `Access-Control-Allow-Origin: *` により、ブラウザのフロントエンド（SPA）から直叩き可能。
- **追加読み込み（ページネーション）完全対応**: 検索・コメント・チャンネル動画で無限スクロール用トークン（`continuation`）を発行。
- **ショート動画（Shorts）特化対応**: 現在の動画に加え、スワイプ用の次動画（3本）を自動抽出。
- **埋め込みプレイヤー即時利用可**: 全動画オブジェクトに `youtube-nocookie.com` の自動再生対応 `embedUrl` を同梱。

---

## 📡 ベース URL

```text
https://<あなたのWorkerサブドメイン>.workers.dev

※ パス形式（/api/v1/...）およびクエリ形式（/?v=... など）の両方に対応しています。

📖 エンドポイント仕様

1. 急上昇・トレンド動画 (/api/v1/trending)

日本国内で現在バズっているトレンド動画・ショートの一覧を取得します。

  - Method: GET
  - Path: /api/v1/trending
  - クエリパラメータ: | パラメータ | 型 | デフォルト | 説明 | | :--- | :--- | :--- | :--- | | type |
    string | default | ジャンル指定 (default: 総合, music: 音楽, gaming: ゲーム, movies: 映画)
    |

レスポンス例

[
  {
    "type": "video",
    "title": "急上昇の動画タイトル",
    "videoId": "xxxxxxxxxxx",
    "author": "チャンネル名",
    "authorId": "UCxxxxxxxxxxxxxxxxxxxxxx",
    "authorUrl": "/channel/UCxxxxxxxxxxxxxxxxxxxxxx",
    "videoThumbnails": [
      {
        "quality": "maxres",
        "url": "https://i.ytimg.com/vi/xxxxxxxxxxx/maxresdefault.jpg",
        "width": 1280,
        "height": 720
      }
    ],
    "viewCount": 1250000,
    "viewCountText": "125万 回視聴",
    "publishedText": "14時間前",
    "lengthSeconds": 345,
    "liveNow": false
  }
]

2. キーワード検索 (/api/v1/search)

指定したキーワードで動画・ショート・プレイリストを横断検索します。

  - Method: GET
  - Path: /api/v1/search
  - クエリパラメータ: | パラメータ | 型 | デフォルト | 説明 | | :--- | :--- | :--- | :--- | | q |
    string | (必須) | 検索キーワード | | limit | number | 30 | 取得件数 (最大 60) | |
    continuation | string | なし | 追加読み込み用トークン (次ページ取得時) |

レスポンス例

{
  "results": [
    {
      "type": "video",
      "isShort": false,
      "title": "通常動画タイトル",
      "videoId": "xxxxxxxxxxx",
      "author": "クリエイター名",
      "authorId": "UCxxxxxxxxxxxxxxxxxxxxxx",
      "authorUrl": "/channel/UCxxxxxxxxxxxxxxxxxxxxxx",
      "videoThumbnails": [ ... ],
      "description": "概要文抜粋...",
      "viewCount": 380000,
      "viewCountText": "38万 回視聴",
      "publishedText": "3日前",
      "lengthSeconds": 240,
      "embedUrl": "https://www.youtube-nocookie.com/embed/xxxxxxxxxxx?autoplay=1&mute=1"
    },
    {
      "type": "short",
      "isShort": true,
      "title": "ショート動画タイトル",
      "videoId": "yyyyyyyyyyy",
      "author": "チャンネル名",
      "videoThumbnails": [ ... ],
      "viewCount": 1200000,
      "lengthSeconds": 30,
      "embedUrl": "https://www.youtube-nocookie.com/embed/yyyyyyyyyyy?autoplay=1&mute=1&controls=0&loop=1&playlist=yyyyyyyyyyy&enablejsapi=1"
    },
    {
      "type": "playlist",
      "title": "プレイリスト名",
      "playlistId": "PLxxxxxxxxxxxxxxxx",
      "author": "作成者名",
      "authorId": "UCxxxxxxxxxxxxxxxxxxxxxx",
      "videoCount": 25,
      "thumbnail": "https://i.ytimg.com/vi/.../hqdefault.jpg"
    }
  ],
  "continuation": "4qmF37qQGmoSI..."
}

3. 通常動画詳細 & 関連動画 (/api/v1/videos/:id)

動画のタイトル、概要欄、チャンネル情報、高評価数、および右側に表示する関連動画一覧を取得します。

  - Method: GET
  - Path: /api/v1/videos/:videoId または /?v=:videoId
  - クエリパラメータ: | パラメータ | 型 | デフォルト | 説明 | | :--- | :--- | :--- | :--- | | limit |
    number | 15 | 関連動画の取得件数 | | continuation | string | なし | 関連動画の追加読み込み用トークン |

レスポンス例

{
  "type": "video",
  "title": "動画タイトル",
  "videoId": "SX_ViT4Ra7k",
  "videoThumbnails": [ ... ],
  "description": "概要欄のテキスト全文...",
  "descriptionHtml": "概要欄（HTMLタグ付き）...",
  "publishedText": "4か月前",
  "viewCount": 38400000,
  "author": "米津玄師",
  "authorId": "UCUCeZaZeJbEYAAzvMgrKOPQ",
  "authorUrl": "/channel/UCUCeZaZeJbEYAAzvMgrKOPQ",
  "authorThumbnails": [
    {
      "url": "https://yt3.ggpht.com/...",
      "width": 48,
      "height": 48
    }
  ],
  "subCountText": "チャンネル登録者数 720万人",
  "embedUrl": "https://www.youtube-nocookie.com/embed/SX_ViT4Ra7k?autoplay=1&mute=1",
  "recommendedVideos": [
    {
      "videoId": "zzzzzzzzzzz",
      "title": "次の関連動画タイトル",
      "author": "関連動画のチャンネル名",
      "authorId": "UC...",
      "authorUrl": "/channel/UC...",
      "lengthSeconds": 195,
      "viewCountText": "120万 回視聴",
      "viewCount": 1200000,
      "videoThumbnails": [ ... ]
    }
  ],
  "continuation": "4qmF37q..."
}

4. ショート動画 & 次動画シーケンス (/api/v1/shorts/:id)

ショート動画のメタデータと、スワイプ時に流れてくる次の動画リスト（3本）を一括取得します。

  - Method: GET
  - Path: /api/v1/shorts/:id または /api/v1/shorts (初期ショート自動選定)
  - クエリパラメータ: | パラメータ | 型 | 説明 | | :--- | :--- | :--- | | sequenceParams |
    string | スワイプ時の追加先読みトークン | | lastVideoId | string |
    直前に再生していたショート動画のID（トークン切れ時の自動復旧用） |

レスポンス例

{
  "status": "success",
  "type": "shorts",
  "current": {
    "id": "9f1rHt0W_Ak",
    "title": "緊急事態発生！不良品",
    "author": "アジーンTV",
    "authorId": "UCxxxxxxxxxxxxxxxxxxxxxx",
    "authorUrl": "/channel/UCxxxxxxxxxxxxxxxxxxxxxx",
    "authorThumbnails": [
      {
        "url": "https://yt3.ggpht.com/...",
        "width": 48,
        "height": 48
      }
    ],
    "likeCount": "1.2万",
    "commentCount": "コメント",
    "embedUrl": "https://www.youtube-nocookie.com/embed/9f1rHt0W_Ak?autoplay=1&mute=1&controls=0&loop=1&playlist=9f1rHt0W_Ak&enablejsapi=1",
    "videoThumbnails": [
      {
        "quality": "vertical",
        "url": "https://i.ytimg.com/vi/9f1rHt0W_Ak/oardefault.jpg",
        "width": 720,
        "height": 1280
      }
    ]
  },
  "sequence": [
    {
      "id": "Hnk48gYYxSQ",
      "title": "だから予想外すぎるって #shorts",
      "author": "チャンネル名",
      "thumbnail": "https://i.ytimg.com/vi/Hnk48gYYxSQ/oardefault.jpg",
      "embedUrl": "https://www.youtube-nocookie.com/embed/Hnk48gYYxSQ?autoplay=1&mute=1&controls=0&loop=1&playlist=Hnk48gYYxSQ&enablejsapi=1"
    },
    {
      "id": "ISHmwrPXjFk",
      "title": "関西の人は、標準語で言えないらしい。",
      "author": "チャンネル名",
      "thumbnail": "https://i.ytimg.com/vi/ISHmwrPXjFk/oardefault.jpg",
      "embedUrl": "https://www.youtube-nocookie.com/embed/ISHmwrPXjFk?autoplay=1&mute=1&controls=0&loop=1&playlist=ISHmwrPXjFk&enablejsapi=1"
    },
    {
      "id": "63CPpM9uvjA",
      "title": "プールの中でうんち💩する人って本当にいるの？",
      "author": "チャンネル名",
      "thumbnail": "https://i.ytimg.com/vi/63CPpM9uvjA/oardefault.jpg",
      "embedUrl": "https://www.youtube-nocookie.com/embed/63CPpM9uvjA?autoplay=1&mute=1&controls=0&loop=1&playlist=63CPpM9uvjA&enablejsapi=1"
    }
  ]
}

5. コメント一覧 (/api/v1/comments/:id)

指定した動画のコメント一覧を取得します。YouTube 最新仕様の entityBatchUpdate に対応し、本文・投稿者名・高評価数が完全に復元されます。

  - Method: GET
  - Path: /api/v1/comments/:videoId または /?comments=:videoId
  - クエリパラメータ: | パラメータ | 型 | デフォルト | 説明 | | :--- | :--- | :--- | :--- | | limit |
    number | 50 | 取得件数 (最大 60) | | continuation | string | なし | コメント追加読み込み用トークン
    |

レスポンス例

{
  "commentCount": 296610,
  "videoId": "SX_ViT4Ra7k",
  "comments": [
    {
      "author": "投稿者ユーザー名",
      "authorUrl": "/channel/UC...",
      "authorId": "UC...",
      "authorThumbnails": [
        {
          "url": "https://yt3.ggpht.com/...",
          "width": 48,
          "height": 48
        }
      ],
      "commentId": "UgxeBLC5VBj2_LdUtrR4AaABAg",
      "authorIsChannelOwner": false,
      "content": "毎朝聴いて元気をもらっています！最高です！",
      "contentHtml": "毎朝聴いて元気をもらっています！最高です！",
      "published": 0,
      "publishedText": "3週間前",
      "likeCount": 352,
      "replyCount": 4,
      "isPinned": false
    }
  ],
  "continuation": "4qmF37qQG..."
}

6. チャンネル情報 & 投稿動画 (/api/v1/channels/:id)

チャンネルの基本情報（バナー、アバター、登録者数、概要欄）と投稿動画一覧を取得します。

  - Method: GET
  - Path: /api/v1/channels/:channelId または /?channel=:channelId
  - 対応ID形式: UCxxxxxxxxxxxxxxxxxxxxxx (Channel ID) または @username (ハンドル名)
  - クエリパラメータ: | パラメータ | 型 | デフォルト | 説明 | | :--- | :--- | :--- | :--- | | limit |
    number | 30 | 動画取得件数 | | continuation | string | なし | 過去動画の追加読み込み用トークン |

レスポンス例

{
  "author": "米津玄師",
  "authorId": "UCUCeZaZeJbEYAAzvMgrKOPQ",
  "authorUrl": "/channel/UCUCeZaZeJbEYAAzvMgrKOPQ",
  "authorThumbnails": [
    {
      "url": "https://yt3.googleusercontent.com/...",
      "width": 512,
      "height": 512
    }
  ],
  "authorBanners": [
    {
      "url": "https://yt3.googleusercontent.com/...",
      "width": 1280,
      "height": 720
    }
  ],
  "subCount": 7200000,
  "description": "米津玄師 公式YouTubeチャンネル",
  "descriptionHtml": "米津玄師 公式YouTubeチャンネル",
  "latestVideos": [
    {
      "type": "video",
      "title": "最新投稿動画タイトル",
      "videoId": "xxxxxxxxxxx",
      "author": "米津玄師",
      "authorId": "UCUCeZaZeJbEYAAzvMgrKOPQ",
      "authorUrl": "/channel/UCUCeZaZeJbEYAAzvMgrKOPQ",
      "videoThumbnails": [ ... ],
      "viewCount": 2400000,
      "viewCountText": "240万 回視聴",
      "publishedText": "2週間前",
      "lengthSeconds": 210,
      "liveNow": false
    }
  ],
  "continuation": "4qmF37q..."
}

7. プレイリスト情報 (/api/v1/playlists/:id)

指定したプレイリストのタイトル、作成者、曲数、および全収録曲リストを取得します。

  - Method: GET
  - Path: /api/v1/playlists/:playlistId または /?playlist=:playlistId
  - 対応ID形式: PLxxxxxxxxxxxxxxxx または VLPLxxxxxxxxxxxxxxxx

レスポンス例

{
  "title": "米津玄師 - MVまとめ",
  "playlistId": "PLlaN88aKQSexv-j9Q9_Ovd2gU_b7I1JbV",
  "author": "作成者名",
  "authorId": "UC...",
  "authorUrl": "/channel/UC...",
  "authorThumbnails": [ ... ],
  "description": "プレイリスト説明文...",
  "descriptionHtml": "プレイリスト説明文...",
  "videoCount": 32,
  "videos": [
    {
      "title": "Lemon",
      "videoId": "SX_ViT4Ra7k",
      "author": "米津玄師",
      "authorId": "UCUCeZaZeJbEYAAzvMgrKOPQ",
      "authorUrl": "/channel/UCUCeZaZeJbEYAAzvMgrKOPQ",
      "videoThumbnails": [ ... ],
      "index": 0,
      "lengthSeconds": 270
    }
  ]
}

🛠️ クライアント実装例 (JavaScript / SPA)

無限スクロールの基本パターン

const API_BASE = "https://あなたのWorker.workers.dev";
let nextToken = null;

// 1. 初回読み込み (50件)
async function fetchInitialVideos(query) {
  const res = await fetch(`${API_BASE}/api/v1/search?q=${encodeURIComponent(query)}&limit=50`);
  const data = await res.json();
  renderVideos(data.results);
  nextToken = data.continuation; // 次ページ用トークンを保管
}

// 2. 最下部到達時の追加読み込み
async function loadMoreVideos() {
  if (!nextToken) return; // 終端なら終了
  const res = await fetch(`${API_BASE}/api/v1/search?continuation=${encodeURIComponent(nextToken)}`);
  const data = await res.json();
  appendVideos(data.results);
  nextToken = data.continuation; // トークンを更新
}

ショート動画スワイプの基本パターン

let currentShort = null;
let queue = [];

// ショートを開く
async function openShort(id) {
  const res = await fetch(`${API_BASE}/api/v1/shorts/${id}`);
  const data = await res.json();
  
  currentShort = data.current;
  queue = data.sequence; // 厳選3本をキューに保持
  
  // 縦型プレイヤーに即時自動再生セット
  document.getElementById("player").src = currentShort.embedUrl;
}

// 下から上にスワイプした時 (次のショートへ)
async function onSwipeNext() {
  if (queue.length === 0) return;
  const nextVideo = queue.shift();
  // 次の動画IDでAPIを再フェッチし、完全なチャンネル名・高評価・新しい次3本を数珠繋ぎ取得
  await openShort(nextVideo.id);
}

🚀 デプロイ手順

1.  Cloudflare Dashboard にログインし、Workers & Pages を選択。
2.  「Create Worker」 をクリック。
3.  提供された api.worker のコードを貼り付け、「Save and deploy」 をクリックします。
4.  生成されたドメイン（https://xxxx.workers.dev）にブラウザでアクセスすると、ビジュアル動作確認用のデモ画面が表示されます。

# ⚠️注意点
1. 自作発言をしないでください
2. また、ここにベタ書きしているapiを使用しないでください
3. streamには対応していません、現存のinvidiousを少しでも軽くするために使用してください
4. 改造をする場合は、chatworkに来て、改造許可を僕にとってください-
