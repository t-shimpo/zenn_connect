---
title: "Go REST API に JWT認可処理を追加する "
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "restapi", "jwt"]
published: true
---
## はじめに
Go の REST API に、**JWTを用いた認可（Authorization）機能**を追加しました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-rest-authorization

## ディレクトリ構成
```bash
myapp
├── auth/
│   ├── context.go           # SetUserID / GetUserID（contextの操作
│   ├── jwt.go               # トークン発行・検証（生成・Parse）
│   └── middleware.go        # JWTミドルウェア（検証 → contextに保存）
├── config/         
│    ├── config.go           # 環境変数の設定など
│    └── db.go               # DB接続設定など
├── handlers/       
│    └── user_handler.go     # HTTPハンドラー(リクエスト受け取り → Serviceを呼び出す)
├── models/         
│    └── user.go             # ドメインモデル定義(User構造体、バリデーションなど)
├── repository/     
│    └── user_repository.go  # DBアクセス処理(UserのCRUD操作)
├── service/        
│    └── user_service.go     # ビジネスロジック(バリデーション、トランザクション制御など)
├── router/         
│    └── router.go           # ハンドラーとルーティングの紐づけ
├── main.go                  # エントリーポイント(Router起動だけにする)
├── .env
├── .env.sample
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── go.mod
└── go.sum
```

## 前提
以下のレポジトのREST APIにJWT認証を追加しました。
https://github.com/t-shimpo/go-rest-jwt
このレポジトリをベースにJWTを使用した認可処理を追加します。

なお、Go・postgreSQLの環境をdocker-composeで構築しています。

## 本記事のゴール
JWT を使った「認証（Authentication）」はすでに導入済みですが、
今回は「認可（Authorization）」＝「**ログイン中のユーザーがアクセス可能な範囲を制限する処理**」を追加します。

ログイン済みのユーザーが、自分自身の情報だけを GET/UPDATE/DELETE できるようにします。

例：
PATCH /users/1 にアクセス
- JWT の sub(subject) が 1 ならOK
- そうでなければ 403 Forbidden

## JWT に含めているsub(userID)をContextから取得する

authパッケージで Context にユーザーIDをセットする関数を実装してあります。
```go:auth/context.go
package auth

import (
	"context"
)

type contextKey string

const UserIDKey contextKey = "userID"

func SetUserID(ctx context.Context, userID int) context.Context {
	return context.WithValue(ctx, UserIDKey, userID)
}
```

Context からユーザーIDを取り出す関数を用意します。
```go:auth/context.go
func GetUserID(ctx context.Context) (int, bool) {
	v := ctx.Value(UserIDKey)
	userID, ok := v.(int)
	return userID, ok
}
```

## ハンドラーで認可チェックを追加

GetUserByID や PatchUser で、
URL の ID と JWT の userID が一致しているかを確認します。

GetUserByID を例として実装します。

既存の実装：
```go:handlers/user_handler.go
func (h *UserHandler) GetUserByID(w http.ResponseWriter, r *http.Request) {
	// URLパスからIDを取得
	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	if idStr == "" {
		respondWithError(w, http.StatusBadRequest, "IDは必要です")
		return
	}

	id, err := strconv.ParseInt(idStr, 10, 64)
	if err != nil {
		respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
		return
	}

	user, err := h.userService.GetUserByID(id)

	// 省略
}
```

以下の流れで認可を行います。
1. r.Context() からユーザーIDを取得
2. URLパラメータのIDを取得
3. 2つか同じか比較
4. 違ったら 403 を返す

```go:handlers/user_handler.go
func (h *UserHandler) GetUserByID(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.GetUserID(r.Context())
	if !ok {
		respondWithError(w, http.StatusUnauthorized, "ログインが必要です")
		return
	}

	// URLパスからIDを取得
	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	if idStr == "" {
		respondWithError(w, http.StatusBadRequest, "IDは必要です")
		return
	}

	id, err := strconv.Atoi(idStr)
	if err != nil {
		respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
		return
	}

	// 認可チェック
	if id != userID {
		respondWithError(w, http.StatusForbidden, "他のユーザーの情報は取得できません")
		return
	}

	// 省略
}
```

これで、GetUserByID に認可処理を組み込むことができました。

## 認可処理を共通化
他の操作にも認可処理を組み込もうと思いますが、
実装としては同じことを行うので共通化しておきます。

```go:handlers/user_handler.go
func authorizeSelf(w http.ResponseWriter, r *http.Request, paramID int) bool {
	userID, ok := auth.GetUserID(r.Context())
	if !ok {
		respondWithError(w, http.StatusUnauthorized, "ログインが必要です")
		return false
	}

	if userID != paramID {
		respondWithError(w, http.StatusForbidden, "他のユーザーの情報は取得できません")
		return false
	}
	return true
}
```

GetUserByID を改善します。
```go:handlers/user_handler.go
func (h *UserHandler) GetUserByID(w http.ResponseWriter, r *http.Request) {
	// URLパスからIDを取得
	idStr := strings.TrimPrefix(r.URL.Path, "/users/")
	if idStr == "" {
		respondWithError(w, http.StatusBadRequest, "IDは必要です")
		return
	}

	id, err := strconv.Atoi(idStr)
	if err != nil {
		respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
		return
	}

	if !authorizeSelf(w, r, id) {
		return
	}

	user, err := h.userService.GetUserByID(id)
	// 省略
}
```

PatchUser、DeleteUser でも同様にauthorizeSelfで認可チェックを行うことができます。

## 動作確認

`/users/:id` で動作確認をします。

#### 正常
ログイン
```bash
$ curl -X POST -i -d '{"email": "taro@example.com" , "password": "pass"}' localhost:8080/login
HTTP/1.1 200 OK
Content-Type: application/json
Date: Sun, 03 Aug 2025 05:31:19 GMT
Content-Length: 164

{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTQyMDI2NzksImlhdCI6MTc1NDE5OTA3OSwiaXNzIjoiIiwic3ViIjoxfQ.q2LM3IuPuRrssquG2EZijpfiik_QGOwCvc2wfZ0jV2c"}
```

GET リクエスト
```bash
$ curl -X GET -i http://localhost:8080/users/1 \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTQyMDI2NzksImlhdCI6MTc1NDE5OTA3OSwiaXNzIjoiIiwic3ViIjoxfQ.q2LM3IuPuRrssquG2EZijpfiik_QGOwCvc2wfZ0jV2c"
HTTP/1.1 200 OK
Content-Type: application/json
Date: Sun, 03 Aug 2025 05:34:30 GMT
Content-Length: 101

{"id":1,"name":"山田太郎","email":"taro@example.com","created_at":"2025-08-03T05:26:55.621008Z"}
```

#### 他人の情報にアクセス
```bash
$ curl -X GET -i http://localhost:8080/users/2 \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTQyMDI2NzksImlhdCI6MTc1NDE5OTA3OSwiaXNzIjoiIiwic3ViIjoxfQ.q2LM3IuPuRrssquG2EZijpfiik_QGOwCvc2wfZ0jV2c"
HTTP/1.1 403 Forbidden
Content-Type: application/json
Date: Sun, 03 Aug 2025 05:34:44 GMT
Content-Length: 64

{"error":"他のユーザーの情報は取得できません"}
```

#### トークンなし
```bash
$ curl -X GET -i http://localhost:8080/users/1
HTTP/1.1 401 Unauthorized
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Sun, 03 Aug 2025 05:29:54 GMT
Content-Length: 33

Authorization header is required
```

## まとめ

- JWTの `sub`（subject）を Context に保存し、ハンドラー内で取得できるようにしました。
- ユーザー自身だけが GET / PATCH / DELETE を実行できるように、認可処理を追加しました。
- 共通関数として `authorizeSelf` を定義することで、他のエンドポイントにも簡単に適用可能です。

次回は、ユーザーの「権限（role）」に応じたアクセス制限などを加えたいと思います。