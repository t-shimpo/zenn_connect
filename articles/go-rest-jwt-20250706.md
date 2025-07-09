---
title: "Go REST API に JWT認証 を組み込む"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "restapi", "jwt"]
published: true
---
## はじめに
Go のアプリケーションにJWT認証機能を作成しました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-rest-jwt

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
以下のレポジトのREST APIにJWT認証を追加する形で実装します。
https://github.com/t-shimpo/go-rest-standard-library-layered

Go・postgreSQLの環境をdocker-composeで構築しています。

## JWT認証の概要
JWT認証の概要を説明します。
詳しい内容は他の記事などを参考にしてください。

### JWT(JSON Web Token)
JWTは
- 誰がこのトークンを発行したか：issuer
- どのユーザーのものか：subject
- 有効期限：expiration

をJSON形式で表し、それを秘密鍵で署名したトークンです。
このトークンを使って認証を行います。
受け取った側は改ざんされていないことを確認でき、中の情報を安全に信用できます。

### 3つのパート
JWTは以下のようにドットで3つに分かれます。
```
xxxxx.yyyyy.zzzzz
```
1. Header
署名方法など (例: HS256)

2. Payload
ユーザー情報や有効期限

3. Signature
Header + PayloadをJWT_SECRETで署名した結果

## 事前準備（パスワードハッシュカラムを追加）
元となるレポジトリの実装ではユーザーにパスワード関連のカラムがないので、
パスワード機能を実装するためにパスワードハッシュカラムを追加しておきます。

マイグレーションを更新します。
```go:config/db.go
func applyMigrations(db *sql.DB) error {
  createUsersTables := `
  CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );`
}
```

User型にPasswordHashを追加します。
ユーザー作成のリクエストでパスワードも受け付けるため、新たにCreateUserRequest型も作成します。
```go:models/user.go
type User struct {
  ID           int       `json:"id"`
  Name         string    `json:"name"`
  Email        string    `json:"email"`
  CreatedAt    time.Time `json:"created_at"`
  PasswordHash string    `json:"-"`
}

type CreateUserRequest struct {
  Name     string `json:"name"`
  Email    string `json:"email"`
  Password string `json:"password"`
}
```

ハンドラー層、サービス層もそれぞれ更新し、
リクエストからパスワードを受け取ってそれをハッシュ化して保存するようにします。
```go:handlers/user_handler.go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
  var req models.CreateUserRequest
  if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    respondWithError(w, http.StatusBadRequest, "無効なリクエストボディ")
    return
  }

  user := models.User{
    Name:  req.Name,
    Email: req.Email,
  }

  createdUser, err := h.userService.CreateUser(&user, req.Password)
    // 省略
}
```

```go:service/user_service.go
func (s *UserService) CreateUser(user *models.User, password string) (*models.User, error) {
  if err := user.Validate(); err != nil {
    return nil, ErrValidation
  }
  if password == "" {
    return nil, ErrValidation
  }

  hashedPassword, err := hashPassword(password)
  if err != nil {
    return nil, err
  }

  user.PasswordHash = hashedPassword

  return s.repo.CreateUser(user)
}

func hashPassword(password string) (string, error) {
  bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
  return string(bytes), err
}
```

## 認証用のトークンを作る仕組みを実装
次に、認証用トークンの生成処理を実装します。

### 環境変数の読み込み
認証トークンの生成のために、秘密鍵などの環境変数を設定します
今回は [godotenv](https://github.com/joho/godotenv) で環境変数を管理しています。

```bash:.env
JWT_SECRET={生成したランダム文字列}  # 例: openssl rand -base64 32 などでランダム文字列を生成
JWT_ISSUER={サービス名}
JWT_EXPIRE_MINUTES={トークン有効期限（分）}
```

環境変数を読み込む処理を追加します。
数字は、文字列→整数に変換しています。
```go:config/config.go
package config

import (
  "os"
  "strconv"
)

var (
  JWTSecret        = os.Getenv("JWT_SECRET")
  JWTIssuer        = os.Getenv("JWT_ISSUER")
  JWTExpireMinutes = getEnvAsInt("JWT_EXPIRE_MINUTES", 60)
)

func getEnvAsInt(key string, defaultValue int) int {
  if valStr := os.Getenv(key); valStr != "" {
    if val, err := strconv.Atoi(valStr); err == nil {
      return val
    }
  }
  return defaultValue
}
```

init関数を追加して、.envを読み込みます。
```go:main.go
func init() {
  if err := godotenv.Load(); err != nil {
    log.Println("環境変数の読み込みに失敗しました:", err)
  }
}

```

### トークン生成
authパッケージを追加して、認証トークン生成処理を実装します。

```go:auth/jwt.go
package auth

import (
  "time"

  "github.com/golang-jwt/jwt/v5"
  "github.com/t-shimpo/go-rest-jwt/config"
)

// 受け取ったユーザーIDからJWTトークンを生成する関数
func GenerateToken(userID int64) (string, error) {
  claims := jwt.MapClaims{
    "sub": userID,                                                                      // subjectとしてユーザーIDを設定
    "iss": config.JWTIssuer,                                                            // 発行者
    "exp": time.Now().Add(time.Duration(config.JWTExpireMinutes) * time.Minute).Unix(), // 有効期限
    "iat": time.Now().Unix(),                                                           // 発行日時
  }

  // HS256で署名するトークンを生成
  token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

  // 秘密鍵を使ってトークンに署名
  return token.SignedString([]byte(config.JWTSecret))
}
```

## ログイン処理を実装
続いてログイン処理を実装します。

ざっくり以下の流れとなります。
1. POST /login
2. リクエストボディをパース
3. DBからユーザーを検索
4. パスワードを検証
5. JWTを生成
6. JSONレスポンスで返す

### パスワード検証処理

サービス層にメールとパスワードを検証する処理を実装します。
ユーザー登録でbcryptを使ったハッシュ化をしているので、同じくbcryptで検証します
```go:service/user_service.go
var (
  ErrValidation        = errors.New("validation error")
  ErrorNotFound        = errors.New("not found")
  ErrorInvalidPassword = errors.New("invalid password")
)
func (s *UserService) Authenticate(email, password string) (*models.User, error) {
  user, err := s.repo.GetUserByEmail(email)
  if err != nil {
    return nil, ErrorNotFound
  }

  if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password)); err != nil {
    return nil, ErrorInvalidPassword
  }
  return user, nil
}
```

```go:repository/user_repository.go
func (r *userRepository) GetUserByEmail(Email string) (*models.User, error) {
  query := `SELECT id, name, email, password_hash, created_at FROM users WHERE email = $1`
  var user models.User

  err := r.db.QueryRow(query, Email).Scan(&user.ID, &user.Name, &user.Email, &user.PasswordHash, &user.CreatedAt)
  if err != nil {
    return nil, err
  }
  return &user, nil
}
```

### ハンドラーにLoginを作成

ログイン処理ではemailとpasswordを受け取るので、これを受ける構造体を用意します。
```go:models/user.go
type LoginRequest struct {
  Email    string `json:"email"`
  Password string `json:"password"`
}
```

ハンドラーにLogin関数を実装します。
リクエストボディをデコードし、パスワード検証を行い、JWTを生成して返却します。
```go:handlers/user_handler.go
func (h *UserHandler) Login(w http.ResponseWriter, r *http.Request) {
  var req models.LoginRequest
  if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    respondWithError(w, http.StatusBadRequest, "無効なリクエストボディ")
    return
  }

  user, err := h.userService.Authenticate(req.Email, req.Password)
  if err != nil {
    if err == service.ErrorNotFound || err == service.ErrorInvalidPassword {
      respondWithError(w, http.StatusUnauthorized, "認証に失敗しました")
      return
    }
    respondWithError(w, http.StatusInternalServerError, "ログイン処理に失敗しました")
    return
  }

  token, err := auth.GenerateToken(int64(user.ID))
  if err != nil {
    respondWithError(w, http.StatusInternalServerError, "トークン生成に失敗しました")
    return
  }

  respondWithJson(w, http.StatusOK, map[string]string{
    "token": token,
  })
}
```

### ルーティングに/loginエンドポイントを追加

```go:router/router.go
func SetupRoutes(userHandler *handlers.UserHandler) *http.ServeMux {
  mux := http.NewServeMux()

    // 省略

  mux.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodPost:
      userHandler.Login(w, r)
    default:
      methodNotAllowedHandler(w)
    }
  })

  return mux
}
```


## JWT認証のミドルウェアを実装

### JWT検証ロジック
トークンの署名・有効期限・issuerを検証する関数を作ります。
```go:auth/jwt.go
// JWTトークンを検証し、Claimsを返す関数
func ParseToken(tokenString string) (jwt.MapClaims, error) {
  token, err := jwt.Parse(tokenString, func(t *jwt.Token) (interface{}, error) {
    // 署名方式の確認
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
      return nil, errors.New("unexpected signing method")
    }
    // 秘密鍵を返す
    return []byte(config.JWTSecret), nil
  })
  if err != nil {
    return nil, err
  }

  // ClaimsをMapClaimsにキャストし、トークンが有効かどうかを確認
  claims, ok := token.Claims.(jwt.MapClaims)
  if !ok || !token.Valid {
    return nil, errors.New("invalid token")
  }

  // 有効期限を検証
  if exp, ok := claims["exp"].(float64); !ok || int64(exp) < time.Now().Unix() {
    return nil, errors.New("token has expired")
  }

  // 発行者を検証
  if iss, ok := claims["iss"].(string); !ok || iss != config.JWTIssuer {
    return nil, errors.New("invalid issuer")
  }

  return claims, nil
}
```

### Contextキー定義

contextを介してログイン済みのユーザーIDを参照できるようにします。
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

### 認証ミドルウェア
認証ミドルウェアを実装します。

まず、`Authorization: Bearer xxx` をパースしてトークンを検証します。
contextにユーザーIDを設定し、次のハンドラーにバトンを渡します。
```go:auth/middleware.go
package auth

import (
  "net/http"
  "strings"
)

func JTWMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // リクエストヘッダーからトークンを取得
    authHeader := r.Header.Get("Authorization")
    if authHeader == "" {
      http.Error(w, "Authorization header is required", http.StatusUnauthorized)
      return
    }
    tokenString := strings.TrimPrefix(authHeader, "Bearer ")

    claims, err := ParseToken(tokenString)
    if err != nil {
      http.Error(w, "Invalid token: "+err.Error(), http.StatusUnauthorized)
      return
    }

    // subject (ユーザーID) を取り出す
    userID, ok := claims["sub"].(float64)
    if !ok {
      http.Error(w, "Invalid token claims", http.StatusUnauthorized)
      return
    }

    // ContextにユーザーIDを設定
    ctx := SetUserID(r.Context(), int(userID))

    // 次のハンドラーを呼び出す
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}
```
## /users/にJWTミドルウェアを適用する
今回は、/users/にJWTミドルウェアを適用します。

ルーターの処理を改善して、auth.JTWMiddleware()を呼び出します。
```go:router/router.go
func SetupRoutes(userHandler *handlers.UserHandler) *http.ServeMux {
  mux := http.NewServeMux()

  // 省略

  protectedHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
      userHandler.GetUserByID(w, r)
    case http.MethodPatch:
      userHandler.PatchUser(w, r)
    case http.MethodDelete:
      userHandler.DeleteUser(w, r)
    default:
      methodNotAllowedHandler(w)
    }
  })

  mux.Handle("/users/", auth.JTWMiddleware(protectedHandler))

  // 省略

  return mux
}
```

### 動作確認
ログインして、ユーザー/users/にアクセスできるか確認します。

**ログイン**
```bash
$ curl -X POST -i -d '{"email": "taro@example.com" , "password": "password"}' localhost:8080/login
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 09 Jul 2025 21:45:38 GMT
Content-Length: 164

{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTIxMDExMzgsImlhdCI6MTc1MjA5NzUzOCwiaXNzIjoiIiwic3ViIjozfQ.2ahtYuBAwoo_-QlqoKVcLNT5CzOcxyRTgJBhZcBbi8w"}
```

**トークンが正しい**
```bash
$ curl -X GET -i http://localhost:8080/users/3 -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3NTIxMDExMzgsImlhdCI6MTc1MjA5NzUzOCwiaXNzIjoiIiwic3ViIjozfQ.2ahtYuBAwoo_-QlqoKVcLNT5CzOcxyRTgJBhZcBbi8w"
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 09 Jul 2025 21:44:29 GMT
Content-Length: 101

{"id":3,"name":"山田太郎","email":"taro@example.com","created_at":"2025-07-02T23:14:39.153998Z"}
```

**トークンが正しくない**
```bash
$ curl -X GET http://localhost:8080/users/3 -H "Authorization: Bearer XXX"
HTTP/1.1 401 Unauthorized
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Wed, 09 Jul 2025 21:44:05 GMT
Content-Length: 64

Invalid token: token signature is invalid: signature is invalid
```