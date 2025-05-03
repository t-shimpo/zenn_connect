---
title: "Go標準パッケージだけで作るREST APIをレイヤー分離する"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "restapi", "postgresql", "docker"]
published: true
---
## はじめに
以前の記事でGo標準ライブラリでREST APIを作りました。
https://zenn.dev/shimpo/articles/go-rest-standard-library-20250406

このコードを元にレイヤーを分離をし、実装を改善しました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-rest-standard-library-layered

### なぜレイヤー分離するのか？
既存の実装では、単一ファイルが中心のシンプルな構成になっています。
ただ、責務があまり分かれていないため、見通しが悪く、拡張性に乏しいです。

これを責務ごとに明確に分離して、
**Handler層 → Service層 → Repository層 → DB** という流れにします。
そうすることにより、機能追加やテストをしやすくし、見通しの良さや拡張性を高めます。

## ディレクトリ構成
ディレクトリ構成は以下のように更新します。

#### 現状のディレクトリ構成
```bash
myapp
├── config
│    └── db.go
├── handlers
│    └── user_handler.go
├── models
│    └── user.go
├── router
│    └── router.go
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── main.go
```

#### 変更後のディレクトリ構成
```bash
myapp/
├── config/         
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
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
```

## POST /users をレイヤー分離する
以下の役割分担を目指して、POST /users をレイヤー分離していきます。
- Handlerは「リクエスト受け取るだけ」
- Serviceは「ビジネスロジック」
- Repositoryは「DBアクセスだけ」
- Modelsは「構造体とバリデーションだけ」

### models/user.go（ドメインモデル）
構造体とバリデーションの実装のみにします。

```go:models/user.go
package models

import "errors"

type User struct {
  ID        int       `json:"id"`
  Name      string    `json:"name"`
  Email     string    `json:"email"`
  CreatedAt time.Time `json:"created_at"`
}

func (u *User) Validate() error {
  if u.Name == "" {
    return errors.New("name is required")
  }
  if u.Email == "" {
    return errors.New("email is required")
  }
  return nil
}
```

### repository/user_repository.go（DBアクセス処理）
データベースに実際にアクセスする処理を書きます。
UserRepositoryインターフェースを定義し、それを満たす構造体を実装しています。

```go:repository/user_repository.go
package repository

import (
  "database/sql"

  "github.com/t-shimpo/go-rest-standard-library-layered/models"
)

type UserRepository interface {
  CreateUser(user *models.User) (*models.User, error)
}

type userRepository struct {
  db *sql.DB
}

func NewUserRepository(db *sql.DB) UserRepository {
  return &userRepository{db: db}
}

func (r *userRepository) CreateUser(user *models.User) (*models.User, error) {
  query := `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, created_at`
  err := r.db.QueryRow(query, user.Name, user.Email).Scan(&user.ID, &user.CreatedAt)
  if err != nil {
    return nil, err
  }
  return user, nil
}
```

### service/user_service.go（ビジネスロジック）
ユースケース単位のロジックを書きます。
Repository のインターフェースに依存します。（中身を知らない）

ここが中心ロジックで、バリデーションや複数処理の組み合わせはここに書きます。

```go:service/user_service.go
package service

import (
  "errors"

  "github.com/t-shimpo/go-rest-standard-library-layered/models"
  "github.com/t-shimpo/go-rest-standard-library-layered/repository"
)

var ErrValidation = errors.New("validation error")

type UserService struct {
  repo repository.UserRepository
}

func NewUserService(repo repository.UserRepository) *UserService {
  return &UserService{repo: repo}
}

func (s *UserService) CreateUser(user *models.User) (*models.User, error) {
  if err := user.Validate(); err != nil {
    return nil, ErrValidation
  }
  return s.repo.CreateUser(user)
}
```

### handlers/user_handler.go（HTTPハンドラー）
HTTPリクエストを処理します。
入力の取り出し → サービスの呼び出し → レスポンス返却 を行います。

```go:handlers/user_handler.go
package handlers

import (
  "encoding/json"
  "net/http"
  "strconv"
  "strings"

  "github.com/t-shimpo/go-rest-standard-library-layered/models"
  "github.com/t-shimpo/go-rest-standard-library-layered/service"
)

type UserHandler struct {
  userService *service.UserService
}

func NewUserHandler(userService *service.UserService) *UserHandler {
  return &UserHandler{userService: userService}
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
  var user models.User
  if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
    respondWithError(w, http.StatusBadRequest, "無効なリクエストボディ")
    return
  }

  createdUser, err := h.userService.CreateUser(&user)
  if err != nil {
    respondWithError(w, http.StatusInternalServerError, "ユーザー作成に失敗しました")
    return
  }

  respondWithJson(w, http.StatusCreated, createdUser)
}

func respondWithJson(w http.ResponseWriter, status int, data interface{}) {
  w.Header().Set("Content-Type", "application/json")
  w.WriteHeader(status)
  json.NewEncoder(w).Encode(data)
}

func respondWithError(w http.ResponseWriter, status int, message string) {
  respondWithJson(w, status, map[string]string{"error": message})
}
```

### router/router.go（ルーティング設定）
```go:router/router.go
package router

import (
  "net/http"

  "github.com/t-shimpo/go-rest-standard-library-layered/handlers"
)

func methodNotAllowedHandler(w http.ResponseWriter) {
  http.Error(w, "許可されていないメソッドです", http.StatusMethodNotAllowed)
}

func SetupRoutes(userHandler *handlers.UserHandler) *http.ServeMux {
  mux := http.NewServeMux()

  mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodPost:
      userHandler.CreateUser(w, r)
    //
    default:
      methodNotAllowedHandler(w)
    }
  })

  //

  return mux
}
```

### config/db.go
この後の main.go の変更のために、InitDB() で *sql.DB を返却するように改善しておきます。

```go:config/db.go
package config

import (
  "database/sql"
  "fmt"
  "os"

  _ "github.com/lib/pq"
)

func InitDB() (*sql.DB, error) {
  dbURL := os.Getenv("DATABASE_URL")
  if dbURL == "" {
    return nil, fmt.Errorf("データベースURLが設定されていません")
  }

  db, err := sql.Open("postgres", dbURL)
  if err != nil {
    return nil, fmt.Errorf("データベース接続に失敗しました: %w", err)
  }

  if err := db.Ping(); err != nil {
    return nil, fmt.Errorf("データベースの接続確認に失敗しました: %w", err)
  }

  if err := applyMigrations(db); err != nil {
    return nil, fmt.Errorf("マイグレーション適用に失敗しました: %w", err)
  }

  return db, nil
}

func applyMigrations(db *sql.DB) error {
  createUsersTables := `
  CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );`

  _, err := db.Exec(createUsersTables)
  if err != nil {
    return err
  }

  fmt.Println("マイグレーションの適用に成功しました")

  return nil
}
```

### main.go（依存関係を注入して起動）
```go:main.go
package main

import (
  "fmt"
  "log"
  "net/http"

  "github.com/t-shimpo/go-rest-standard-library-layered/config"
  "github.com/t-shimpo/go-rest-standard-library-layered/handlers"
  "github.com/t-shimpo/go-rest-standard-library-layered/repository"
  "github.com/t-shimpo/go-rest-standard-library-layered/service"

  "github.com/t-shimpo/go-rest-standard-library-layered/router"
)

func main() {
  // DB 初期化
  db, err := config.InitDB()
  if err != nil {
    fmt.Println("DB初期化エラー:", err)
    return
  }
  defer db.Close()

  userRepo := repository.NewUserRepository(db)
  userService := service.NewUserService(userRepo)
  userHandler := handlers.NewUserHandler(userService)
  mux := router.SetupRoutes(userHandler)

  fmt.Println("サーバーが 8080 ポートで起動中")
  log.Fatal(http.ListenAndServe(":8080", mux))
}
```

POST /users をレイヤー分離することができました。

## そのほかのAPIをレイヤー分離する
その他のAPIも同様に、レイヤー分離しました。
詳しくは以下のレポジトリをご覧ください。
https://github.com/t-shimpo/go-rest-standard-library-layered

## 最後に
今回はPOST /users の実装にフォーカスしてレイヤー分離を行いました。
小さなAPIでも、レイヤーを分けることで保守性が高まり、機能追加もスムーズになります。
さらにコードの見通しが良くなったと感じます。

今後は、トランザクション制御やミドルウェア導入など、さらに実践的な内容にも取り組んでいきたいと思います。