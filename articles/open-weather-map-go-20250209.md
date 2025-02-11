---
title: "OpenWeatherMap APIを使う"
emoji: "🌤"
type: "tech"
topics: ["openweather", "api" ]
published: false
---

## はじめに
### OpenWeatherMap APIとは？
世界中の天気予報を取得できるAPIで、現在の天気、予報、過去の気象データなどを取得可能です。
**リアルタイムの天気情報**だけでなく、**気象履歴や大気汚染データ**なども取得できます。

📌 **公式サイト**
https://openweathermap.org/

※2025年2月時点の情報です。

### 本記事の目的
本記事では、以下のことを解説します。
- OpenWeatherMap APIの使い方
- いくつかのエンドポイントの紹介

## OpenWeatherMap API の概要
### APIの種類
OpenWeatherMap では、以下のようなAPIを提供しています。
- **Current Weather API**（現在の天気）
- **5 Day Weather Forecast API**（5日間の3時間ごとの予報）
- **Weather Maps API**（天気のマップデータ）
- **Air Pollution API**（大気汚染情報）
- **Geocoding API**（都市名から緯度・経度を取得）

これらのAPIは、アカウント登録後に **APIキーを取得することで利用可能** です。
他にも、サブスクリプションを登録することで利用できる専門的なAPIがあります。

📌 **公式ドキュメント**: [OpenWeatherMap API](https://openweathermap.org/api)


### 無料枠のルール
上記のAPIは、以下の制限のもと無料で利用できます。
- **最大 60 API calls / 分**
- **最大 1,000,000 API calls / 月**

また、[One Call API 3.0](https://openweathermap.org/api/one-call-3) (詳細な気象情報・予報) も、サブスクリプション登録により以下の条件で利用できます。
- **1日 1,000回まで無料**
- **超過分は 1リクエストあたり 0.0012 GBP（約0.22円）の課金**


📌 **料金詳細**: [OpenWeatherMap Pricing](https://openweathermap.org/price)

## OpenWeatherMap APIの登録方法
### アカウント登録
1. 公式サイトへアクセス
[OpenWeatherMap](https://openweathermap.org/) の公式サイトにアクセスします。

2. アカウント作成
右上の **Sign in** から **Create an Account** を選択し、必要情報を入力してアカウントを作成します。
登録したメールアドレスに確認メールが届くので、認証を完了させます。

3. APIキーの取得
右上の自身のユーザーネームをクリックし、メニューから**My API keys** をクリックします。
![](/images/open-weather-map-go-20250209/api_key_1.png)
デフォルトで発行されているAPIキーを確認します。
![](/images/open-weather-map-go-20250209/api_key_2.png)
必要に応じて **Create key**から新しいAPIキーを発行できます。

## APIリクエストを実行する
いくつかのAPIで東京のデータを取得してみます。
- 緯度: 35.689
- 経度: 139.692

### リクエスト方法
`https://api.openweathermap.org/data/2.5/weather?lat=35.6895&lon=139.692&appid={API_KEY}` のように appid パラメータにAPIキーを指定します。

### APIレスポンスの主なデータ項目
各APIのレスポンスには、以下のようなデータが含まれます。

|キー|説明|
|--|--|
|coord|位置情報（緯度・経度）|
|weather|天候情報（天気ID、概要、詳細、アイコン）|
|main|気温（K）、体感温度、気圧、湿度|
|visibility|視界の良さ（メートル単位）|
|wind|風速（m/s）、風向き（°）|
|clouds|雲量（%）|
|dt|データ取得時刻（UNIXタイムスタンプ）|
|sys|国コード、日の出・日の入り時刻|
|timezone|タイムゾーン（秒単位のオフセット）|

※ 気温（main.temp）はデフォルトでケルビン（K）表示なので、摂氏（℃）に変換する場合は temp - 273.15 を計算する

### Current Weather API（現在の天気）
📌 ドキュメント: https://openweathermap.org/current

#### パラメータ
- `lat`: Latitude(緯度)
- `lon`: Longitude(経度)
#### リクエスト
```sh
curl "https://api.openweathermap.org/data/2.5/weather?lat=35.6895&lon=139.692&appid={API_KEY}"
```
#### レスポンス
```json
{
	"coord": {
		"lon": 139.6917,
		"lat": 35.6895
	},
	"weather": [
		{
			"id": 801,
			"main": "Clouds",
			"description": "few clouds",
			"icon": "02d"
		}
	],
	"base": "stations",
	"main": {
		"temp": 281.63,
		"feels_like": 277.52,
		"temp_min": 280.9,
		"temp_max": 282.73,
		"pressure": 1020,
		"humidity": 34,
		"sea_level": 1020,
		"grnd_level": 1018
	},
	"visibility": 10000,
	"wind": {
		"speed": 9.26,
		"deg": 340
	},
	"clouds": {
		"all": 20
	},
	"dt": 1739242385,
	"sys": {
		"type": 2,
		"id": 268395,
		"country": "JP",
		"sunrise": 1739223169,
		"sunset": 1739261897
	},
	"timezone": 32400,
	"id": 1850144,
	"name": "Tokyo",
	"cod": 200
}
```

### 5 Day Weather Forecast API（5日間の3時間ごとの予報）
📌 ドキュメント: https://openweathermap.org/forecast5

#### パラメータ
- `lat`: Latitude(緯度)
- `lon`: Longitude(経度)
#### リクエスト
```sh
curl "https://api.openweathermap.org/data/2.5/forecast?lat=35.6895&lon=139.692&appid={API_KEY}"
```
#### レスポンス
```json
{
	"cod": "200",
	"message": 0,
	"cnt": 40,
	"list": [
		{
			"dt": 1739253600,
			"main": {
				"temp": 281.76,
				"feels_like": 278.1,
				"temp_min": 281.76,
				"temp_max": 282.46,
				"pressure": 1020,
				"sea_level": 1020,
				"grnd_level": 1018,
				"humidity": 28,
				"temp_kf": -0.7
			},
			"weather": [
				{
					"id": 801,
					"main": "Clouds",
					"description": "few clouds",
					"icon": "02d"
				}
			],
			"clouds": {
				"all": 20
			},
			"wind": {
				"speed": 7.7,
				"deg": 342,
				"gust": 7.17
			},
			"visibility": 10000,
			"pop": 0,
			"sys": {
				"pod": "d"
			},
			"dt_txt": "2025-02-11 06:00:00"
		}
                 // 中略
	],
	"city": {
		"id": 1850144,
		"name": "Tokyo",
		"coord": {
			"lat": 35.6895,
			"lon": 139.6917
		},
		"country": "JP",
		"population": 12445327,
		"timezone": 32400,
		"sunrise": 1739223169,
		"sunset": 1739261897
	}
}
```

### Geocoding API（都市名から緯度・経度を取得）
📌 ドキュメント: https://openweathermap.org/api/geocoding-api

#### パラメータ
- `q`: 都市名や国コード
#### リクエスト
```sh
curl "https://api.openweathermap.org/geo/1.0/direct?q=Tokyo&limit=2&appid={API_KEY}"
```

#### レスポンス
```json
[
	{
		"name": "Tokyo",
		"local_names": {
			"ia": "Tokyo",
			"pl": "Tokio",
			"bg": "Токио",
			"da": "Tokyo",
			"hu": "Tokió",
			"kn": "ಟೋಕ್ಯೊ",
			"ku": "Tokyo",
			"hr": "Tokio",
			"sl": "Tokio",
			"be": "Токіа",
			"lt": "Tokijas",
			"he": "טוקיו",
			"mr": "तोक्यो",
			"fr": "Tokyo",
                        // 中略

		},
		"lat": 35.6828387,
		"lon": 139.7594549,
		"country": "JP"
	},
	{
		"name": "Chofu",
		"local_names": {
			"pl": "Chōfu",
			"ar": "تشوفو، طوكيو",
			"sr": "Чофу",
			"fa": "چوفو، توکیو",
			"fr": "Chōfu",
                        // 中略
		},
		"lat": 35.660036,
		"lon": 139.554815,
		"country": "JP"
	}
]
```

## One Call API 3.0
**One Call API 3.0**では詳細な気象データを取得できます。

### サブスクリプションの登録
**One Call API 3.0**を使用するためにはサブスクリプションの登録が必要です。
[OpenWeatherMap Pricing](https://openweathermap.org/price)から支払い方法などを登録します。
![](/images/open-weather-map-go-20250209/one_call_api.png)

以下の条件で利用できます。
- **1日 1,000回まで無料**
- **超過分は 1リクエストあたり 0.0012 GBP（約0.22円）の課金**

### エンドポイント
One Call API 3.0 には、以下のようなエンドポイントがあります。

|エンドポイント|概要|
|---|---|
|Current weather and forecasts|現在の天気と天気予報（分単位・時間単位・日単位）|
|Weather data for any timestamp|任意の日時の天気データを取得（過去・未来の天気）|
|Daily aggregation|日ごとの気象データ（最高気温・最低気温・降水量など）|
|Weather overview|1つのリクエストで複数地点の天気情報を取得|

📌 **ドキュメント**: [One Call API 3.0](https://openweathermap.org/api/one-call-3)

以下で **Current weather and forecasts**を叩いています。

### Current weather and forecasts（現在の天気と予報）
#### パラメータ
- `lat`: Latitude(緯度)
- `lon`: Longitude(経度)

`exclude`で除外する項目をしていることも可能です。

#### リクエスト
```sh
curl "https://api.openweathermap.org/data/3.0/onecall?lat=35.6895&lon=139.692&appid={API key}"
```
#### レスポンス

分単位の予報`minutely`、時間単位の予報`hourly`、日単位の予報`daily`の取得も可能です。
気象警報データもレスポンスされます。

```json
{
	"lat": 35.6895,
	"lon": 139.692,
	"timezone": "Asia/Tokyo",
	"timezone_offset": 32400,
	"current": {
		"dt": 1739260426,
		"sunrise": 1739223169,
		"sunset": 1739261897,
		"temp": 280.58,
		"feels_like": 276.42,
		"pressure": 1022,
		"humidity": 28,
		"dew_point": 264.35,
		"uvi": 0,
		"clouds": 20,
		"visibility": 10000,
		"wind_speed": 8.23,
		"wind_deg": 10,
		"weather": [
			{
				"id": 801,
				"main": "Clouds",
				"description": "few clouds",
				"icon": "02d"
			}
		]
	},
    "minutely": [ 
    // 中略
    ],
	"hourly": [
        {
			"dt": 1739257200,
			"temp": 280.86,
			"feels_like": 276.84,
			"pressure": 1022,
			"humidity": 27,
			"dew_point": 264.16,
			"uvi": 0.29,
			"clouds": 18,
			"visibility": 10000,
			"wind_speed": 8.03,
			"wind_deg": 340,
			"wind_gust": 7.17,
			"weather": [
				{
					"id": 801,
					"main": "Clouds",
					"description": "few clouds",
					"icon": "02d"
				}
			],
			"pop": 0
		},
                // 中略
    ],
	"daily": [
		{
			"dt": 1739239200,
			"sunrise": 1739223169,
			"sunset": 1739261897,
			"moonrise": 1739256660,
			"moonset": 1739220720,
			"moon_phase": 0.45,
			"summary": "Expect a day of partly cloudy with clear spells",
			"temp": {
				"day": 280.55,
				"min": 277.27,
				"max": 281.97,
				"night": 277.44,
				"eve": 280.58,
				"morn": 277.55
			},
			"feels_like": {
				"day": 275.95,
				"night": 273.46,
				"eve": 276.31,
				"morn": 274.44
			},
			"pressure": 1021,
			"humidity": 27,
			"dew_point": 262.68,
			"wind_speed": 10.63,
			"wind_deg": 339,
			"wind_gust": 13.45,
			"weather": [
				{
					"id": 800,
					"main": "Clear",
					"description": "clear sky",
					"icon": "01d"
				}
			],
			"clouds": 0,
			"pop": 0,
			"uvi": 3.32
		}
	],
	"alerts": [
		{
			"sender_name": "JMA",
			"event": "乾燥注意報",
			"start": 1739199601,
			"end": 1739286000,
			"description": "",
			"tags": [
				"Air quality"
			]
		}
	]
}
```