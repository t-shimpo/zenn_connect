---
title: "GoでSpotifyの新作リリース情報を配信するLINE Botを実装する"
emoji: "🎧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "linebot", "spotify"]
published: true
---

[Spotify Web API](https://developer.spotify.com/documentation/web-api/)を使って、最新のリリース情報を配信するLINE BotをGoで実装しました。

なお、作成したものについて、以下の記事で[Render](https://render.com/)の**Cron Jobs**使って定期的に配信するようにしました。
https://zenn.dev/shimpo/articles/run-line-bot-with-render-cron

## 作ったもの
以下のようにSpotifyの新作リリース情報を配信するLINE Botを作成しました。
![](/images/spotify-new-release-bot/line-screen1.png)

今回の実装したアプリは下記レポジトリで公開しています。
https://github.com/t-shimpo/spotify-new-release-bot

## 使用技術
- Go
- Line Message API
- Spotify Web API

## 事前準備
### LINE Messaging API
LINE Messaging APIを使用するために、以下の手順で準備をします。

#### LINE Developersコンソールにログイン
[こちら](https://developers.line.biz/ja/docs/line-developers-console/login-account/)の手順に従ってログインを行なってください。

#### チャンネルを作成
[こちら](https://developers.line.biz/ja/docs/messaging-api/getting-started/)を参考にチャンネルを作成します。

#### チャネルシークレットの取得
チャンネルを作成したら、作成したチャンネルのページの`チャネル基本設定`の項目の中の
**チャネルシークレット**の値を確認します。
この値は、後述する**チャンネルアクセストークン**と併せて、認証に必要となります。

#### チャネルアクセストークンの発行
作成したチャンネルのページの`Messaging API設定`のタグをクリックします。
**チャネルアクセストークン**の項目の`発行`ボタンをクリックします。
発行した値を後の実装で使用します。

※トークンの概要については、[こちら](https://developers.line.biz/ja/docs/messaging-api/channel-access-tokens/)をご覧ください。

#### 友だち追加
動作確認のために、作成したチャンネル（ボット）を友だちとして追加します。
作成したチャンネルのページの`Messaging API設定`のタグのクリックすると**QRコード**の項目がありますので、LINEで読み取り、友だち追加します。

### Spotify Web API
Spotify Web APIを使用するために、Spotifyアカウントのセットアップ・クライアントアプリケーションの作成が必要です。
また、作成したアプリの**Client ID**と**Client Secret**の情報も必要です。

以下の記事にそれらの設定の手順を記載してありますので、ご覧ください。
https://zenn.dev/shimpo/articles/trying-spotify-api-with-go

## 実装

### 使用するライブラリ
#### Line Message APIの操作
以下の公式SDKが手提供されておりますので、使用します。

https://github.com/line/line-bot-sdk-go

#### Spotify Web APIの操作
Spotifyのデータの取得には以下のライブラリを使用します。
[APIエンドポイントリファレンス](https://developer.spotify.com/documentation/web-api/reference/#/)の全てを叩けるようにサポートされているライブラリです。

https://github.com/zmb3/spotify

#### その他
今回は、認可で使用する値を`.env`ファイルで環境変数として扱うために、godotenvライブラリも使用しました。

https://github.com/joho/godotenv

以下のような形で実装できます。

```:.env
LINE_CHANNEL_SECRET=XXX  # チャネルシークレット
LINE_CHANNEL_TOKEN=XXX   # チャネルアクセストークン
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
	secret := os.Getenv("LINE_CHANNEL_SECRET")
	token := os.Getenv("LINE_CHANNEL_TOKEN")

	fmt.Println(secret)
	fmt.Println(token)
}
```

### LINE BotでHello Worldを送信
まずはLINE Botで簡単なメッセージを送信してみます。
前述したLINE Messaging API SDKをインストールします。
https://github.com/line/line-bot-sdk-go
```
$ go get github.com/line/line-bot-sdk-go/v7/linebot
```

また、前述したgodotenvライブラリもインストールします。
https://github.com/joho/godotenv
```
$ go get github.com/joho/godotenv
```

Messaging APIでのメッセージの送り方にはいくつかの種類があります。
今回は、全ての友だちに送信する**ブロードキャストメッセージ**の方法で送ります。
また、メッセージタイプはテキストメッセージです。
メッセージについて、詳しくは公式ドキュメントの[こちら](https://developers.line.biz/ja/docs/messaging-api/sending-messages/)に記載されています。

以下のように実装します。

```:.env
LINE_CHANNEL_SECRET=XXX  # チャネルシークレット
LINE_CHANNEL_TOKEN=XXX   # チャネルアクセストークン
```

```go:main.go
package main

import (
	"log"
	"os"

	"github.com/joho/godotenv"
	"github.com/line/line-bot-sdk-go/v7/linebot"
)

func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	bot, err := linebot.New(
		os.Getenv("LINE_CHANNEL_SECRET"),
		os.Getenv("LINE_CHANNEL_TOKEN"),
	)
	if err != nil {
		log.Fatal(err)
	}

	message := linebot.NewTextMessage("Hello World")

	if _, err := bot.BroadcastMessage(message).Do(); err != nil {
		log.Fatal(err)
	}
}
```
送信するメッセージは、[NewTextMessage](https://pkg.go.dev/github.com/line/line-bot-sdk-go/v7/linebot#NewTextMessage)関数で生成することができ、
それを`bot.BroadcastMessage(message).Do()`と呼び出すことで友だち登録しているユーザー全員に送信できます。

**LINE画面**
![](/images/spotify-new-release-bot/line-screen2.png)
「Hello World」と送信することができました。

### Spotify APIから新作リリース情報を取得する
#### 事前準備
Spotify Web APIを使用するために、Spotifyアカウントのセットアップとクライアントアプリケーションの作成が必要です。
また、作成したアプリの**Client ID**と**Client Secret**の情報も必要です。
設定がまだの方は、[こちらの記事](https://zenn.dev/shimpo/articles/trying-spotify-api-with-go)を参考に設定を行なってください。

認可については、記事と同様で、今回はユーザーの情報にはアクセスしないので、
[Client credentials](https://developer.spotify.com/documentation/general/guides/authorization/client-credentials/)で行います。

#### 実装
Spotify Web APIの操作を行うために、以下のライブラリをインストールします。
https://github.com/zmb3/spotify

```
$ go get github.com/line/line-bot-sdk-go/v7/linebot
```

そして以下のように実装することで、新作リリース情報を取得できます。

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

	res, err := client.NewReleases(ctx, spotify.Country("JP"), spotify.Limit(10))
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(res)
}
```

[NewReleases](https://pkg.go.dev/github.com/zmb3/spotify/v2#Client.NewReleases)という関数が用意されていますので、これを使用します。
第一引数にcontext、第二引数以降で、オプションを渡すことができます。
オプションはCountry、Limit、 Offsetを指定できます。

Countryを指定することで、指定した国に即した情報を取得できます。
なお、取得対象が日本のアーティストに限定されるわけではありません。
ISO 3166-1 alpha-2 の二文字のコードを使用して指定します。日本は`JP`です。
（参考：[ISO 3166 Country Codes](https://www.iso.org/iso-3166-country-codes.html)）

今回は、Countryを日本に設定し、また、10個のアルバムデータを取得するためにLimitをしていしました。

### 新作リリース情報をBotから送信する

Spotify APIから取得した新作リリース情報をLINE Botで送信します。
以下のように実装しました。

```:.env
LINE_CHANNEL_SECRET=XXX  # チャネルシークレット
LINE_CHANNEL_TOKEN=XXX   # チャネルアクセストークン

SPOTIFY_ID=XXX       # Client ID
SPOTIFY_SECRET=XXX   # Client Secret
```

```go:main.go
package main

import (
	"context"
	"log"
	"os"

	spotifyauth "github.com/zmb3/spotify/v2/auth"

	"github.com/joho/godotenv"
	"github.com/line/line-bot-sdk-go/v7/linebot"
	"github.com/zmb3/spotify/v2"
	"golang.org/x/oauth2/clientcredentials"
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

	res, err := client.NewReleases(ctx, spotify.Country("US"), spotify.Limit(10))
	if err != nil {
		log.Fatal(err)
	}

	bot, err := linebot.New(
		os.Getenv("LINE_CHANNEL_SECRET"),
		os.Getenv("LINE_CHANNEL_TOKEN"),
	)
	if err != nil {
		log.Fatal(err)
	}

	firstMessage := linebot.NewTextMessage("新作リリース情報をお届けします！")
	if _, err := bot.BroadcastMessage(firstMessage).Do(); err != nil {
		log.Fatal(err)
	}

	var messages []linebot.SendingMessage
	// NOTE: 1回のリクエストで送信できるメッセージオブジェクトは5つ
	for _, v := range res.Albums {
		messages = append(messages, linebot.NewTextMessage(v.ExternalURLs["spotify"]))
		if len(messages) == 5 {
			if _, err := bot.BroadcastMessage(messages...).Do(); err != nil {
				log.Fatal(err)
			}
			messages = nil
		}
	}
}
```
1回のリクエストで送れるメッセージオブジェクトは最大5つなので、
forループでスライスに格納し、5つずつ送信する形にしました。

**LINE画面**
![](/images/spotify-new-release-bot/line-screen1.png)

リリース情報を送信できました。

## RenderのCron Jobsで定期実行する
[Render](https://render.com/)の**Cron Jobs**使って定期的に実行し、毎週特定の時間にリリース情報を配信するようにします。

環境変数の設定をRender側で行うため、環境変数の取得周りを微修正します。
Render側で`GO_ENV`という環境変数を扱う前提で、その文字列が`production`でない場合にのみ、`.env`から環境変数をロードするとして、開発環境・本番環境ともに正常に動作するよう改善します。

```diff go:main.go
+	if os.Getenv("GO_ENV") != "production" {
		err := godotenv.Load()
		if err != nil {
			log.Fatal("Error loading .env file")
		}
+	}
```

Render側の設定は以下の記事に書きましたので、参考にしていただければと思います。
https://zenn.dev/shimpo/articles/run-line-bot-with-render-cron

## 参考資料
https://developers.line.biz/ja/docs/messaging-api/

https://developers.line.biz/ja/docs/line-developers-console/login-account/

https://developer.spotify.com/documentation/web-api/

https://www.iso.org/iso-3166-country-codes.html

https://zenn.dev/shimpo/articles/trying-spotify-api-with-go
