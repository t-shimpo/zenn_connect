---
title: "GoでSpotify Web APIを叩いてみる "
emoji: "🎧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "spotify", ]
published: false
---
Spotifyが公開している[Web API](https://developer.spotify.com/documentation/web-api/)をGoを使って叩いてみました。

## APIの概要
[Spotify Web API](https://developer.spotify.com/documentation/web-api/)はRESTfulなWeb APIで、アーティスト・アルバム・楽曲などに関するJSONメタデータを取得できます。
また、ユーザーがYour Musicライブラリに保存したプレイリストや音楽など、ユーザーに関連するデータへのアクセスも提供します。
本記事では、アーティスト情報や楽曲情報の取得のみを扱い、ユーザーデータへのアクセスは扱いません。

## 事前準備
#### アカウントのセットアップ
- Spotifyユーザーアカウントの作成
  - アカウントがなければ[こちら](https://open.spotify.com/)より作成します（無料のアカウントで大丈夫です）
- Spotify for Developersの[ダッシュボード](https://developer.spotify.com/dashboard/)にログイン
  - 作成したSpotifyユーザーアカウントでログインをします
  - 利用規約に同意する必要があります

#### クライアントアプリケーションを作成
※ドキュメントは[こちら](https://developer.spotify.com/documentation/general/guides/authorization/app-settings/)

ダッシュボードにログインし、以下を行います。
1. `CREATE AN APP`ボタンをクリック
![](/images/trying-spotify-api-with-go/create-app-image1.png)

2. モーダルに必要事項を入力し`CREATE`ボタンをクリック
![](/images/trying-spotify-api-with-go/create-app-image2.png)

アプリが作成されたら、作成されたアプリのページの画面左上部にある**Client ID**と、その下の`SHOW CLIENT SECRET`をクリックすると表示される**Client Secret**の値を確認できます。
これら2つの情報は、APIを叩く際に必要となります。
後の実装で環境変数としてこれらの値を扱います。

## 認可について
※ドキュメントは[こちら](https://developer.spotify.com/documentation/general/guides/authorization/)

Spotifyのデータや機能にアクセスするために認可のプロセスが必要です。
その方法として、4つの方法が用意されていますが、今回はユーザーの情報にはアクセスしないので、[Client credentials](https://developer.spotify.com/documentation/general/guides/authorization/client-credentials/)で行います。

## APIを叩いてみる
### 使用するライブラリ
Spotify APIを叩くために、以下のライブラリを使用します。
https://github.com/zmb3/spotify
[APIエンドポイントリファレンス](https://developer.spotify.com/documentation/web-api/reference/#/)の全てを叩けるようにサポートされているライブラリです。
具体的な実装は後述します。

また、今回は、認可で使用する値を.envファイルで環境変数として扱うために、godotenvライブラリも使用しました。
https://github.com/joho/godotenv

**Client ID**の値を`SPOTIFY_ID`に、**Client Secret**の値を`SPOTIFY_SECRET`にそれぞれ入れます。

以下のような形で実装できます。

```:.env
SPOTIFY_ID=XXX       # Client ID
SPOTIFY_SECRET=XXX   # Client Secret
```

```go:main.go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/joho/godotenv"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}
	id := os.Getenv("SPOTIFY_ID")
	secret := os.Getenv("SPOTIFY_SECRET")

	fmt.Println(id)
	fmt.Println(secret)
}

```

### 認可周りの実装
今回は、[Client credentials](https://developer.spotify.com/documentation/general/guides/authorization/client-credentials/)の認可を行うため、以下のように実装します。

```:.env
SPOTIFY_ID=XXX       # Client ID
SPOTIFY_SECRET=XXX   # Client Secret
```

```go:main.go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	spotifyauth "github.com/zmb3/spotify/v2/auth"

	"golang.org/x/oauth2/clientcredentials"

	"github.com/joho/godotenv"
	"github.com/zmb3/spotify/v2"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	ctx := context.Background()
	config := &clientcredentials.Config{
		ClientID:     os.Getenv("SPOTIFY_ID"),
		ClientSecret: os.Getenv("SPOTIFY_SECRET"),
		TokenURL:     spotifyauth.TokenURL,
	}
	token, err := config.Token(ctx)
	if err != nil {
		log.Fatalf("couldn't get token: %v", err)
	}

	httpClient := spotifyauth.New().Client(ctx, token)
	client := spotify.New(httpClient)

	var id spotify.ID = "07w0rG5TETcyihsEIZR3qG"
	result, err := client.GetAlbum(ctx, id)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(result)
}
```

### 検索する
[APIリファレンス](https://developer.spotify.com/documentation/web-api/reference/#/operations/search)
キーワード文字列に一致するアルバム、アーティスト、プレイリスト、楽曲などの情報を取得できます。
Search関数の引数に`SearchTypeArtist`や`SearchTypePlaylist`を渡すことで、検索するアイテムの種類を指定できます。

```go:main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	spotifyauth "github.com/zmb3/spotify/v2/auth"

	"golang.org/x/oauth2/clientcredentials"

	"github.com/joho/godotenv"
	"github.com/zmb3/spotify/v2"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	ctx := context.Background()
	config := &clientcredentials.Config{
		ClientID:     os.Getenv("SPOTIFY_ID"),
		ClientSecret: os.Getenv("SPOTIFY_SECRET"),
		TokenURL:     spotifyauth.TokenURL,
	}
	token, err := config.Token(ctx)
	if err != nil {
		log.Fatalf("couldn't get token: %v", err)
	}

	httpClient := spotifyauth.New().Client(ctx, token)
	client := spotify.New(httpClient)

	result, err := client.Search(ctx, "Beatles", spotify.SearchTypeArtist, spotify.Limit(2))
	if err != nil {
		log.Fatal(err)
	}

	bytes, _ := json.MarshalIndent(result, "", "  ")
	fmt.Println(string(bytes))
}

```
**出力**
```json
{
  "artists": {
    "href": "https://api.spotify.com/v1/search?query=Beatles\u0026type=artist\u0026offset=0\u0026limit=2",
    "limit": 2,
    "offset": 0,
    "total": 106,
    "next": "https://api.spotify.com/v1/search?query=Beatles\u0026type=artist\u0026offset=2\u0026limit=2",
    "previous": "",
    "items": [
      {
        "name": "The Beatles",
        "id": "3WrFJ7ztbogyGnTHbHJFl2",
        "uri": "spotify:artist:3WrFJ7ztbogyGnTHbHJFl2",
        "href": "https://api.spotify.com/v1/artists/3WrFJ7ztbogyGnTHbHJFl2",
        "external_urls": {
          "spotify": "https://open.spotify.com/artist/3WrFJ7ztbogyGnTHbHJFl2"
        },
        "popularity": 83,
        "genres": [
          "beatlesque",
          "british invasion",
          "classic rock",
          "merseybeat",
          "psychedelic rock",
          "rock"
        ],
        "followers": {
          "total": 24103862,
          "href": ""
        },
        "images": [
          {
            "height": 640,
            "width": 640,
            "url": "https://i.scdn.co/image/ab6761610000e5ebe9348cc01ff5d55971b22433"
          },
          {
            "height": 320,
            "width": 320,
            "url": "https://i.scdn.co/image/ab67616100005174e9348cc01ff5d55971b22433"
          },
          {
            "height": 160,
            "width": 160,
            "url": "https://i.scdn.co/image/ab6761610000f178e9348cc01ff5d55971b22433"
          }
        ]
      },
      {
        "name": "beatles",
        "id": "3YojAkU7hiAalEunJy55JW",
        "uri": "spotify:artist:3YojAkU7hiAalEunJy55JW",
        "href": "https://api.spotify.com/v1/artists/3YojAkU7hiAalEunJy55JW",
        "external_urls": {
          "spotify": "https://open.spotify.com/artist/3YojAkU7hiAalEunJy55JW"
        },
        "popularity": 11,
        "genres": [],
        "followers": {
          "total": 3059,
          "href": ""
        },
        "images": []
      }
    ]
  },
  "albums": null,
  "playlists": null,
  "tracks": null,
  "shows": null,
  "episodes": null
}
```

### 関連するアーティストを取得
[APIリファレンス](https://developer.spotify.com/documentation/web-api/reference/#/operations/get-an-artists-related-artists)
あるアーティストに関連するアーティストのリストを取得できます。

```go:main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	spotifyauth "github.com/zmb3/spotify/v2/auth"

	"golang.org/x/oauth2/clientcredentials"

	"github.com/joho/godotenv"
	"github.com/zmb3/spotify/v2"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	ctx := context.Background()
	config := &clientcredentials.Config{
		ClientID:     os.Getenv("SPOTIFY_ID"),
		ClientSecret: os.Getenv("SPOTIFY_SECRET"),
		TokenURL:     spotifyauth.TokenURL,
	}
	token, err := config.Token(ctx)
	if err != nil {
		log.Fatalf("couldn't get token: %v", err)
	}

	httpClient := spotifyauth.New().Client(ctx, token)
	client := spotify.New(httpClient)

	// The BeatlesのSpotify IDを指定
	result, err := client.GetRelatedArtists(ctx, "3WrFJ7ztbogyGnTHbHJFl2")
	if err != nil {
		log.Fatal(err)
	}

	bytes, _ := json.MarshalIndent(result[0:2], "", "  ")
	fmt.Println(string(bytes))
}

```
**出力**
```json
[
  {
    "name": "John Lennon",
    "id": "4x1nvY2FN8jxqAFA0DA02H",
    "uri": "spotify:artist:4x1nvY2FN8jxqAFA0DA02H",
    "href": "https://api.spotify.com/v1/artists/4x1nvY2FN8jxqAFA0DA02H",
    "external_urls": {
      "spotify": "https://open.spotify.com/artist/4x1nvY2FN8jxqAFA0DA02H"
    },
    "popularity": 74,
    "genres": [
      "album rock",
      "art rock",
      "beatlesque",
      "classic rock",
      "mellow gold",
      "rock"
    ],
    "followers": {
      "total": 5160600,
      "href": ""
    },
    "images": [
      {
        "height": 640,
        "width": 640,
        "url": "https://i.scdn.co/image/ab6761610000e5eb207c6849d1a1f4480e6aa222"
      },
      {
        "height": 320,
        "width": 320,
        "url": "https://i.scdn.co/image/ab67616100005174207c6849d1a1f4480e6aa222"
      },
      {
        "height": 160,
        "width": 160,
        "url": "https://i.scdn.co/image/ab6761610000f178207c6849d1a1f4480e6aa222"
      }
    ]
  },
  {
    "name": "George Harrison",
    "id": "7FIoB5PHdrMZVC3q2HE5MS",
    "uri": "spotify:artist:7FIoB5PHdrMZVC3q2HE5MS",
    "href": "https://api.spotify.com/v1/artists/7FIoB5PHdrMZVC3q2HE5MS",
    "external_urls": {
      "spotify": "https://open.spotify.com/artist/7FIoB5PHdrMZVC3q2HE5MS"
    },
    "popularity": 67,
    "genres": [
      "album rock",
      "beatlesque",
      "classic rock",
      "folk rock",
      "mellow gold",
      "rock",
      "singer-songwriter",
      "soft rock"
    ],
    "followers": {
      "total": 2303804,
      "href": ""
    },
    "images": [
      {
        "height": 640,
        "width": 640,
        "url": "https://i.scdn.co/image/ab6761610000e5eb7fda8adcbe0cd99ec83a3bcf"
      },
      {
        "height": 320,
        "width": 320,
        "url": "https://i.scdn.co/image/ab676161000051747fda8adcbe0cd99ec83a3bcf"
      },
      {
        "height": 160,
        "width": 160,
        "url": "https://i.scdn.co/image/ab6761610000f1787fda8adcbe0cd99ec83a3bcf"
      }
    ]
  }
]
```

### Audio Features（楽曲の詳細な情報）を取得する
[APIリファレンス](https://developer.spotify.com/documentation/web-api/reference/#/operations/get-audio-features)
Audio Featuresという楽曲の詳細な情報を取得できる、ユニークなエンドポイントもあります。
テンポなどに加え、acousticness、danceabilltyなどのデータが取得できます。

```go:main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	spotifyauth "github.com/zmb3/spotify/v2/auth"

	"golang.org/x/oauth2/clientcredentials"

	"github.com/joho/godotenv"
	"github.com/zmb3/spotify/v2"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	ctx := context.Background()
	config := &clientcredentials.Config{
		ClientID:     os.Getenv("SPOTIFY_ID"),
		ClientSecret: os.Getenv("SPOTIFY_SECRET"),
		TokenURL:     spotifyauth.TokenURL,
	}
	token, err := config.Token(ctx)
	if err != nil {
		log.Fatalf("couldn't get token: %v", err)
	}

	httpClient := spotifyauth.New().Client(ctx, token)
	client := spotify.New(httpClient)

	searchResult, err := client.Search(ctx, "In My Life", spotify.SearchTypeTrack)
	if err != nil {
		log.Fatal(err)
	}

	trackId := searchResult.Tracks.Tracks[0].ID
	feature, err := client.GetAudioFeatures(ctx, trackId)
	if err != nil {
		log.Fatal(err)
	}

	bytes, _ := json.MarshalIndent(feature[0], "", "  ")
	fmt.Println(string(bytes))
}

```
**出力**
```json
{
  "acousticness": 0.449,
  "analysis_url": "https://api.spotify.com/v1/audio-analysis/3KfbEIOC7YIv90FIfNSZpo",
  "danceability": 0.688,
  "duration_ms": 146333,
  "energy": 0.435,
  "id": "3KfbEIOC7YIv90FIfNSZpo",
  "instrumentalness": 0,
  "key": 9,
  "liveness": 0.113,
  "loudness": -11.359,
  "mode": 1,
  "speechiness": 0.0323,
  "tempo": 103.239,
  "time_signature": 4,
  "track_href": "https://api.spotify.com/v1/tracks/3KfbEIOC7YIv90FIfNSZpo",
  "uri": "spotify:track:3KfbEIOC7YIv90FIfNSZpo",
  "valence": 0.435
}
```

## 参考資料
[https://developer.spotify.com/documentation/web-api/](https://developer.spotify.com/documentation/web-api/)
[https://developer.spotify.com/documentation/general/guides/authorization/](https://developer.spotify.com/documentation/general/guides/authorization/)
