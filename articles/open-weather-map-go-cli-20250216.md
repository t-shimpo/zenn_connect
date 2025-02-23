---
title: "GoでOpenWeatherMap APIを利用したCLIツールを作成する"
emoji: "⛅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openweather", "api", "go", "cli"]
published: false
---

## はじめに
### 本記事の目的
- Go言語を用いて、OpenWeatherMap APIを利用したCLIツールを作成します。
- 指定した都市の「現在の天気」または「5日間の3時間毎の天気予報」を取得できます。

なお、作成したものは、以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-open-weather-cli

### 前提
- Goがインストールされていること
- OpenWeatherMap APIのAPIキーを取得済みであること

OpenWeatherMap APIについての紹介、APIキーの取得方法、今回使用するAPIの解説などは以下の記事にまとめておりますのでご覧ください。
https://zenn.dev/shimpo/articles/open-weather-map-go-20250209

## CLIツールの仕様
### コマンド例
```sh
weather current tokyo    # 現在の天気を取得
weather forecast tokyo   # 5日間の3時間毎の天気予報を取得
```

### 実装する機能
1. **都市名から緯度・経度を取得（Geocoding API）**
2. **現在の天気を取得（Current Weather API）**
3. **5日間の天気予報を取得（5 Day Forecast API）**

## ディレクトリ構成
```
myapp
├── api
│    ├── current_weather.go
│    ├── geocode.go
│    └── weather_forecast.go
├── config
│    └── config.go
├── .env
├── .gitignore
├── go.mod
├── go.sum
└── main.go
```

## プロジェクトのセットアップ
`go mod init`でGoモジュールを作成します。
```:go.mod
module github.com/t-shimpo/go-weather-cli

go 1.24.0
```


### CLIプログラムの作成
まず、都市名を受け取り出力する最小限のCLIプログラムを作成します。

```go:main.go
package main

import (
	"fmt"
	"os"
)

func main() {
	if len(os.Args) < 4 || os.Args[1] != "weather" {
		fmt.Println("Usage: weather [current|forecast] <city>")
		os.Exit(1)
	}

	command := os.Args[2]
	city := os.Args[3]

	switch command {
	case "current":
		fmt.Printf("Fetching current weather for %s\n", city)
		// ここに現在の天気を取得する処理を追加
	case "forecast":
		fmt.Printf("Fetching weather forecast for %s\n", city)
		// ここに天気予報を取得する処理を追加
	default:
		fmt.Println("Invalid command. Use 'current' or 'forecast'.")
		os.Exit(1)
	}
}

```

`weather current <city>`、`weather forecast <city>`のコマンドを受け付けて出力をするCLIプログラムができました。

**実行結果**
```
$ go run main.go weather forecast tokyo
Fetching weather forecast for tokyo
```

## 環境変数の管理
OpenWeatherMap APIのリクエストでAPIキーを使用するので、[godotenv](https://github.com/joho/godotenv?tab=readme-ov-file)を使用し、`.env`ファイルで環境変数を管理します。
APIキーの取得方法については[こちらの記事](https://github.com/t-shimpo/go-open-weather-cli)をご覧ください。

APIキーを取得する関数を追加しました。
```go:config/config.go
package config

import (
	"fmt"
	"os"

	"github.com/joho/godotenv"
)

func GetAPIKey() (string, error) {
	_ = godotenv.Load()
	apiKey := os.Getenv("OPEN_WEATHER_API_KEY")

	if apiKey == "" {
		return "", fmt.Errorf("OPEN_WEATHER_API_KEY is not set")
	}

	return apiKey, nil
}
```
`.env`ファイルに以下のように設定してください
```:.env
OPEN_WEATHER_API_KEY={API_KEY}
```

main関数で以下のようにAPIキーを取得して使用します。
```go:main.go
package main

import (
	"fmt"
	"os"

	"github.com/t-shimpo/go-weather-cli/config"
)

func main() {
	apiKey, err := config.GetAPIKey()
	if err != nil {
		fmt.Println("Error: ", err)
		os.Exit(1)
	}

    // 中略
}
```


## Geocoding API を使って都市名を緯度・経度に変換する
続いて、都市名を経度・緯度に変換する処理を実装します。
**Current Weather API**、**5 Day Forecast API**　のリクエストのパラメータとして経度・緯度のデータが必要になるため、**Geocoding API**を使用してデータを取得します。

📌 APIドキュメント: https://openweathermap.org/api/geocoding-api

`GetCoordinates`関数を追加しました。
```go:api/geocode.go
package api

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type GeocodeResponse struct {
	Lat float64 `json:"lat"`
	Lon float64 `json:"lon"`
}

func GetCoordinates(city string, apiKey string) (float64, float64, error) {
	url := fmt.Sprintf("https://api.openweathermap.org/geo/1.0/direct?q=%s&limit=1&appid=%s", city, apiKey)
	resp, err := http.Get(url)
	if err != nil {
		return 0, 0, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return 0, 0, fmt.Errorf("api error: status code %d", resp.StatusCode)
	}

	var result []GeocodeResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return 0, 0, err
	}

	if len(result) == 0 {
		return 0, 0, fmt.Errorf("no results")
	}

	return result[0].Lat, result[0].Lon, nil
}

```

```diff go:main.go
import (
	"fmt"
	"os"

+	"github.com/t-shimpo/go-weather-cli/api"
	"github.com/t-shimpo/go-weather-cli/config"
)

func main() {
	apiKey, err := config.GetAPIKey()
	if err != nil {
		fmt.Println("Error: ", err)
		os.Exit(1)
	}

        // 中略

	command := os.Args[2]
	city := os.Args[3]

+	lat, lon, err := api.GetCoordinates(city, apiKey)
+	if err != nil {
+		fmt.Println("Error: ", err)
+		os.Exit(1)
+	}
+	fmt.Printf("Coordinates for %s: %f, %f\n", city, lat, lon)

        // 中略
}
```

データが取得できることを確認します。
```
$ go run main.go weather forecast tokyo
Coordinates for tokyo: 35.682839, 139.759455
Fetching weather forecast for tokyo
```

## 現在の天気を取得（Current Weather API）
続いて、取得した緯度・経度の情報から**Current Weather API** で現在の天気を取得する処理を実装します。

📌 APIドキュメント: https://openweathermap.org/current

`GetCurrentWeather`関数を追加しました。
```go:api/current_weather.go
package api

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type CurrentWeatherResponse struct {
	Weather []struct {
		Description string `json:"description"`
	} `json:"weather"`
	Main struct {
		Temp     float64 `json:"temp"`
		Humidity float64 `json:"humidity"`
	} `json:"main"`
	Wind struct {
		Speed float64 `json:"speed"`
	} `json:"wind"`
}

func GetCurrentWeather(lat float64, lon float64, apiKey string) (string, error) {
	url := fmt.Sprintf("https://api.openweathermap.org/data/2.5/weather?lang=ja&units=metric&lat=%f&lon=%f&appid=%s", lat, lon, apiKey)
	resp, err := http.Get(url)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return "", fmt.Errorf("api error: status code %d", resp.StatusCode)
	}

	var result CurrentWeatherResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return "", err
	}

	str := fmt.Sprintf("天気: %s\n気温: %.2f ℃\n湿度: %.2f %%\n風速: %.2f m/s",
		result.Weather[0].Description, result.Main.Temp, result.Main.Humidity, result.Wind.Speed)
	return str, nil
}
```
APIリクエストのパラメータに含まれる`lang`,`units`は任意です。
`lang`を指定することで、`weather.description`を日本語で取得することが可能です。
`units`を指定することで`main.temp`摂氏の気温を取得できます。

```diff go:main.go
func main() {
    // 中略

-	fmt.Printf("Coordinates for %s: %f, %f\n", city, lat, lon)

	switch command {
	case "current":
-		fmt.Printf("Fetching current weather for %s\n", city)
-		// ここに現在の天気を取得する処理を追加
+		result, err := api.GetCurrentWeather(lat, lon, apiKey)
+		if err != nil {
+			fmt.Println("Error: ", err)
+			os.Exit(1)
+		}
+		fmt.Println(result)
	case "forecast":
		fmt.Printf("Fetching weather forecast for %s\n", city)
		// ここに天気予報を取得する処理を追加
	default:
		fmt.Println("Invalid command. Use 'current' or 'forecast'.")
		os.Exit(1)
	}
}
```

データが取得できることを確認します。
```
$ go run main.go weather current tokyo
天気: 薄い雲
気温: 6.05℃
湿度: 28.00 %
風速: 11.32 m/s
```

## 5日間の3時間毎の天気予報を取得（5 Day Forecast API）
続いて、取得した緯度・経度の情報から**5 Day Forecast API** で5日間の3時間毎の天気予報を取得する処理を実装します。

📌 APIドキュメント: https://openweathermap.org/forecast5

`GetWeatherForecast`関数を追加しました。
```go:api/current_weather.go
package api

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

type WeatherForecastResponse struct {
	List []struct {
		DtTxt   string `json:"dt_txt"`
		Weather []struct {
			Description string `json:"description"`
		} `json:"weather"`
		Main struct {
			Temp float64 `json:"temp"`
		} `json:"main"`
	} `json:"list"`
}

func GetWeatherForecast(lat float64, lon float64, apiKey string) (string, error) {
	url := fmt.Sprintf("https://api.openweathermap.org/data/2.5/forecast?lang=ja&units=metric&lat=%f&lon=%f&appid=%s", lat, lon, apiKey)
	resp, err := http.Get(url)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return "", fmt.Errorf("api error: status code %d", resp.StatusCode)
	}

	var result WeatherForecastResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return "", err
	}

	str := "3時間ごとの天気予報\n"

	jst, _ := time.LoadLocation("Asia/Tokyo")

	for _, forecast := range result.List {
		// 日付フォーマットを整形
		t, _ := time.Parse("2006-01-02 15:04:05", forecast.DtTxt) // UTC でパース
		t = t.In(jst)                                             // JST に変換
		formattedTime := fmt.Sprintf("%d月%d日 %02d:%02d", t.Month(), t.Day(), t.Hour(), t.Minute())

		str += fmt.Sprintf("%s  %s %.2f ℃\n", formattedTime, forecast.Weather[0].Description, forecast.Main.Temp)
	}

	return str, nil
}
```

```diff go:main.go
func main() {
    // 中略

	switch command {
	case "current":
		result, err := api.GetCurrentWeather(lat, lon, apiKey)
		if err != nil {
			fmt.Println("Error: ", err)
			os.Exit(1)
		}
		fmt.Println(result)
	case "forecast":
-		fmt.Printf("Fetching weather forecast for %s\n", city)
-		// ここに天気予報を取得する処理を追加
+		result, err := api.GetWeatherForecast(lat, lon, apiKey)
+		if err != nil {
+			fmt.Println("Error: ", err)
+			os.Exit(1)
+		}
+		fmt.Println(result)
	default:
		fmt.Println("Invalid command. Use 'current' or 'forecast'.")
		os.Exit(1)
	}
}
```

データが取得できることを確認します。
```
$ go run main.go weather forecast tokyo
3時間ごとの天気予報
2月23日 00:00  晴天 3.06 ℃
2月23日 03:00  晴天 3.02 ℃
2月23日 06:00  薄い雲 2.55 ℃
2月23日 09:00  曇りがち 3.95 ℃
2月23日 12:00  曇りがち 5.75 ℃
2月23日 15:00  曇りがち 6.60 ℃
2月23日 18:00  厚い雲 6.20 ℃
2月23日 21:00  厚い雲 6.03 ℃
2月24日 00:00  雲 5.40 ℃
2月24日 03:00  薄い雲 3.76 ℃
2月24日 06:00  曇りがち 2.71 ℃
2月24日 09:00  厚い雲 4.06 ℃
2月24日 12:00  晴天 7.82 ℃
2月24日 15:00  雲 8.53 ℃
...
```