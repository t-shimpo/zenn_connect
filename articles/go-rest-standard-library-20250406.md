---
title: "Go標準ライブラリでREST APIを作る"
emoji: "🐹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "restapi", "postgresql", "docker"]
published: true
---
## はじめに
Go の 標準ライブラリで REST API を作成しました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-rest-standard-library

## ディレクトリ構成
```
myapp
├── config
│    └── db.go
├── handlers
│    └── user_handler.go  # User 構造体の定義とDB操作
├── models
│    └── user.go          # ユーザー関連の API 処理
├── router
│    └── router.go        # ルーティング設定
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── main.go
```

## 前提
Go・postgreSQLの環境をdocker-composeで構築しています。
本記事では環境構築方法については省略しますが、下記の記事で解説していますので、参考にしてください。
https://zenn.dev/shimpo/articles/go-postgres-docker-20250316

## テーブル設計
今回はユーザー情報のテーブルとそれを操作するAPIを作成することにしました。

ユーザー情報の項目
- `id` (`int`, 主キー, 自動採番)
- `name` (`string`, 必須)
- `email` (`string`, 必須, 一意)
- `created_at` (`timestamp`, デフォルト: 現在時刻)

## usersテーブルの作成
まずは、データベースにusersテーブルを作成します。
実務上はマイグレーションを管理するツールや仕組みを導入するかと思いますが、
今回は簡易的にusersテーブル作成のマイグレーションを実行します。

`config/db.go`にデータベースの初期化処理を実装します。
データベースへの接続を確認してから、`users`テーブルを作成するクエリを実行しています。

```go:config/db.go
package config

import (
    "database/sql"
    "fmt"
    "os"

    _ "github.com/lib/pq"
)

var DB *sql.DB

func InitDB() error {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return fmt.Errorf("データベースURLが設定されていません")
    }

    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        return fmt.Errorf("データベース接続に失敗しました: %w", err)
    }

    if err := db.Ping(); err != nil {
        return fmt.Errorf("データベースの接続確認に失敗しました: %w", err)
    }

    DB = db

    if err := applyMigrations(); err != nil {
        return fmt.Errorf("マイグレーション適用に失敗しました: %w", err)
    }

    return nil
}

func applyMigrations() error {
    createUsersTables := `
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );`

    _, err := DB.Exec(createUsersTables)
    if err != nil {
        return err
    }

    fmt.Println("マイグレーションの適用に成功しました")

    return nil
}
```

```go:main.go
package main

import (
    "fmt"

    "github.com/t-shimpo/go-rest-standard-library/config"
)

func main() {
    // DB 初期化
    err := config.InitDB()
    if err != nil {
        fmt.Println("DB初期化エラー:", err)
        return
    }

    if config.DB != nil {
        defer func() {
            if err := config.DB.Close(); err != nil {
                fmt.Println("DBクローズエラー:", err)
            }
        }()
    }
}
```

なお、データベースURLは`DATABASE_URL`として取得できるように docker-compose に設定しています。
```yml:docker-compose.yml
services:
  app:
    //
    environment:
      DATABASE_URL: postgres://user:password@db:5432/mydb?sslmode=disable
```

### マイグレーションの実行を確認
以下のコマンドを実行し、postgreSQLコンテナに入ります。
```
$ docker exec -it postgres psql -U user -d mydb
```

`\dt;` `\d users;`でテーブルが作成されていることを確認できます。
```
mydb=# \dt;
       List of relations
 Schema | Name  | Type  | Owner 
--------+-------+-------+-------
 public | users | table | user
(1 row)

mydb=# \d users;
                                        Table "public.users"
   Column   |            Type             | Collation | Nullable |              Default              
------------+-----------------------------+-----------+----------+-----------------------------------
 id         | integer                     |           | not null | nextval('users_id_seq'::regclass)
 name       | text                        |           | not null | 
 email      | text                        |           | not null | 
 created_at | timestamp without time zone |           |          | CURRENT_TIMESTAMP
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

## POST /users API
まず、ユーザー作成API `POST /users` を実装します。

### User 構造体とデータ操作
`models/user.go`にユーザー構造体と、データ操作の関数を作成します。
`CreateUser`では`RETURNING`を指定して、挿入したデータをそのまま取得しています。

`QueryRow`は単一行の結果のみを期待するクエリの実行に使用されます。
`Scan`で構造体に値を割り当てます。

```go:models/user.go
package models

import (
    "fmt"
    "time"

    "github.com/t-shimpo/go-rest-standard-library/config"
)

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func CreateUser(name, email string) (*User, error) {
    query := `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email, created_at`
    user := &User{}

    err := config.DB.QueryRow(query, name, email).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    if err != nil {
        return nil, fmt.Errorf("ユーザーの作成に失敗しました: %w", err)
    }

    return user, nil
}
```

### ハンドラー
POST リクエストを受け取る HTTP ハンドラー を作ります。

まず、`json.NewDecoder(r.Body).Decode(&req)` でリクエストのJSONを構造体に変換しています。
その後、必須フィールドのバリデーションを行っています。
今回は、簡易的にnilや空文字でなければOKとしています。

`CreateUser`を実行して成功した場合は、成功レスポンス `201 Created` を返しています。

```go:handlers/user_handler.go
package handlers

import (
    "encoding/json"
    "net/http"
    "strings"

    "github.com/t-shimpo/go-rest-standard-library/models"
)

type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

func respondWithJson(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondWithError(w http.ResponseWriter, status int, message string) {
    respondWithJson(w, status, map[string]string{"error": message})
}

// `POST /users`
func CreateUserHandler(w http.ResponseWriter, r *http.Request) {

    var req CreateUserRequest
    defer r.Body.Close()

    // json デコード
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondWithError(w, http.StatusBadRequest, "無効なリクエストボディ")
        return
    }

    // バリデーション
    req.Name = strings.TrimSpace(req.Name)
    req.Email = strings.TrimSpace(req.Email)

    if req.Name == "" {
        respondWithError(w, http.StatusBadRequest, "名前は必須です")
        return
    }

    if req.Email == "" {
        respondWithError(w, http.StatusBadRequest, "メールは必須です")
        return
    }

    // DB に保存
    user, err := models.CreateUser(req.Name, req.Email)
    if err != nil {
        respondWithError(w, http.StatusInternalServerError, "ユーザー作成中にエラーが発生しました")
        return
    }

    respondWithJson(w, http.StatusCreated, user)
}
```

### ルーティング
続いて、ルーティングの設定です。
`CreateUserHandler` を `POST /users` に紐付けます。

```go:router/router.go
package router

import (
    "net/http"

    "github.com/t-shimpo/go-rest-standard-library/handlers"
)

func methodNotAllowedHandler(w http.ResponseWriter) {
    http.Error(w, "許可されていないメソッドです", http.StatusMethodNotAllowed)
}

func SetupRoutes() *http.ServeMux {
    mux := http.NewServeMux()

    mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodPost:
            handlers.CreateUserHandler(w, r)
        default:
            methodNotAllowedHandler(w)
        }
    })

    return mux
}
```

`main.go`にサーバーを起動する処理を実装します。

```go:main.go
import (
    "fmt"
    "log"
    "net/http"

    "github.com/t-shimpo/go-rest-standard-library/config"
    "github.com/t-shimpo/go-rest-standard-library/router"
)

func main() {
    // DB 初期化
    err := config.InitDB()
    if err != nil {
        fmt.Println("DB初期化エラー:", err)
        return
    }

    if config.DB != nil {
        defer func() {
            if err := config.DB.Close(); err != nil {
                fmt.Println("DBクローズエラー:", err)
            }
        }()
    }

    // ルーティング設定
    mux := router.SetupRoutes()

    // サーバー起動
    fmt.Println("サーバーが 8080 ポートで起動中")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### 動作確認
正常系
```
$ curl -X POST -i -d '{"name": "山田太郎", "email": "taro@example.com"}' localhost:8080/users
HTTP/1.1 201 Created
Content-Type: application/json
Date: Tue, 01 Apr 2025 16:16:39 GMT
Content-Length: 101

{"id":1,"name":"山田太郎","email":"test@example.com","created_at":"2025-03-30T23:08:59.364493Z"}
```

異常系
```
$ curl -X POST -i -d '{"name": "鈴木りょう", "email": ""}' localhost:8080/users
HTTP/1.1 400 Bad Request
Content-Type: application/json
Date: Tue, 01 Apr 2025 16:16:39 GMT
Content-Length: 37

{"error":"メールは必須です"}
```


## GET /users/{id} API
続いて、ユーザー取得API `GET /users/{id}` を実装します。

### データ操作
先ほどと同様、`QueryRow().Scan()` を使用して実装しています。
`QueryRow().Scan()` は「該当なし」なら `sql.ErrNoRows` を返します。
該当なしの場合`404 Not Found`を返却したいので、別のエラーと分けています。

```go:models/user.go
func GetUserByID(id int) (*User, error) {
    query := `SELECT id, name, email, created_at FROM users WHERE id = $1`
    user := &User{}

    err := config.DB.QueryRow(query, id).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("sql: no rows in result set") // ユーザーが見つからない場合
        }
        return nil, fmt.Errorf("ユーザーの取得に失敗しました: %w", err) // その他エラー
    }

    return user, nil
}
```

### ハンドラー
GET リクエストを受け取る HTTP ハンドラー を作ります。
`TrimPrefix`で URL から ID を取得し、整数に変換してから`GetUserByID`を実行しています。
`sql: no rows in result set`を受け取った場合は`404 Not Found`を返却しています。

```go:handlers/user_handler.go
// `GET /users/{id}`
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    // URL から ID を取得
    idStr := strings.TrimPrefix(r.URL.Path, "/users/")
    if idStr == "" {
        respondWithError(w, http.StatusBadRequest, "IDが必要です")
        return
    }

    // ID を整数に変換
    id, err := strconv.Atoi(idStr)
    if err != nil {
        respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
        return
    }

    // DB からユーザー取得
    user, err := models.GetUserByID(id)
    if err != nil {
        if err.Error() == "sql: no rows in result set" {
            respondWithError(w, http.StatusNotFound, "ユーザーが見つかりません")
        } else {
            respondWithError(w, http.StatusInternalServerError, "ユーザー取得中にエラーが発生しました")
        }
        return
    }

    respondWithJson(w, http.StatusOK, user)
}
```

### ルーティング
`GetUserByID` を `GET /users/` に紐付けます。

```go:router/router.go
func SetupRoutes() *http.ServeMux {
    mux := http.NewServeMux()

    //

    mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            handlers.GetUserHandler(w, r)
        default:
            methodNotAllowedHandler(w)
        }
    })

    return mux
}
```

### 動作確認
正常系
```
$ curl -X GET -i http://localhost:8080/users/1
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 01 Apr 2025 19:16:39 GMT
Content-Length: 101

{"id":1,"name":"山田太郎","email":"test@example.com","created_at":"2025-03-30T23:08:59.364493Z"}
```

異常系
```
$ curl -X GET -i http://localhost:8080/users/abc
HTTP/1.1 400 Bad Request
Content-Type: application/json
Date: Tue, 01 Apr 2025 19:16:22 GMT
Content-Length: 54

{"error":"IDは数値である必要があります"}
```


## GET /users API
続いて、ユーザー一覧取得API `GET /users` を実装します。

### データ操作
`limit`と`offset`も受け取れるように実装しました。
`Query`を実行し、クエリの結果から1行ずつユーザー構造体に値を割り当てています。
`rows.Next()` のループ中には検出されないエラーもあるため、`rows.Err()`で最終的なエラーもチェックしています。

```go:models/user.go
func GetUsers(limit, offset int) ([]User, error) {
    query := `SELECT id, name, email, created_at FROM users ORDER BY id LIMIT $1 OFFSET $2`
    rows, err := config.DB.Query(query, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("ユーザー一覧の取得に失敗しました: %w", err)
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt); err != nil {
            return nil, fmt.Errorf("ユーザーのデータ取得中にエラーが発生しました: %w", err)
        }
        users = append(users, user)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("ユーザー一覧の取得中にエラーが発生しました: %w", err)
    }

    return users, nil
}
```

### ハンドラー
GET リクエストを受け取る HTTP ハンドラー を作ります。
クエリパラメータから`limit`と`offset`を取得し、整数に変換して`GetUsers`に渡しています。

```go:handlers/user_handler.go
// `GET /users`
func GetUsersHandler(w http.ResponseWriter, r *http.Request) {
    limit, err := strconv.Atoi(r.URL.Query().Get("limit"))
    if err != nil || limit <= 0 {
        limit = 10 // デフォルト値
    }

    offset, err := strconv.Atoi(r.URL.Query().Get("offset"))
    if err != nil || offset < 0 {
        offset = 0 // デフォルト値
    }

    users, err := models.GetUsers(limit, offset)
    if err != nil {
        respondWithError(w, http.StatusInternalServerError, "ユーザー取得中にエラーが発生しました")
        return
    }

    respondWithJson(w, http.StatusOK, users)
}
```

### ルーティング
`GetUsers` を `GET /users` に紐付けます。

```go:router/router.go
func SetupRoutes() *http.ServeMux {
    mux := http.NewServeMux()

    mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodPost:
            handlers.CreateUserHandler(w, r)
        case http.MethodGet:
            handlers.GetUsersHandler(w, r)
        default:
            methodNotAllowedHandler(w)
        }
    })

    //

    return mux
}
```

### 動作確認
正常系
```
$ curl -X GET -i http://localhost:8080/users
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 02 Apr 2025 22:20:03 GMT
Content-Length: 469

[{"id":1,"name":"山田太郎","email":"test@example.com","created_at":"2025-03-30T23:08:59.364493Z"},{"id":2,"name":"青木ただよし","email":"test_aoki@example.com","created_at":"2025-03-30T23:10:58.343966Z"}]
```
異常系
```
$ curl -X GET -i "http://localhost:8080/users?limit=1&offset=1"
HTTP/1.1 200 OK
Content-Type: application/json
Date: Wed, 02 Apr 2025 22:22:03 GMT
Content-Length: 114

[{"id":2,"name":"青木ただよし","email":"test_aoki@example.com","created_at":"2025-03-30T23:10:58.343966Z"}]
```


## PATCH /users/{id} API
続いて、ユーザー情報を更新するAPI `PATCH /users/{id}` を実装します。

### データ操作
`name` `email` はそれぞれ指定されない（nil）の可能性を考慮して`*string`型にしています。
そして、nilでないフィールドだけUPDATEするようにします。

`setClauses`に更新対象のカラムだけ追加し、`args`に値を入れる形で、SET句を動的に構築しています。

```go:models/user.go
func UpdateUser(id int, name, email *string) (*User, error) {
    updates := map[string]interface{}{}

    if name != nil {
        updates["name"] = *name
    }

    if email != nil {
        updates["email"] = *email
    }

    // SET句を動的に構築
    setClauses := []string{}
    args := []interface{}{}
    argIndex := 1

    for column, value := range updates {
        setClauses = append(setClauses, fmt.Sprintf("%s = $%d", column, argIndex))
        args = append(args, value)
        argIndex++
    }

    query := fmt.Sprintf(
        "UPDATE users SET %s WHERE id = $%d RETURNING id, name, email, created_at",
        strings.Join(setClauses, ", "), argIndex,
    )
    args = append(args, id)

    user := &User{}
    err := config.DB.QueryRow(query, args...).Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("sql: no rows in result set")
        }
        return nil, fmt.Errorf("ユーザーの更新に失敗しました: %w", err)
    }

    return user, nil
}
```

### ハンドラー
PATCH リクエストを受け取る HTTP ハンドラー を作ります。

Name、Emailのいずれもリクエストボディに含まれていない場合はエラーを発生させています。
また、今回も`sql: no rows in result set`を受け取った場合は`404 Not Found`を返却しています。

```go:handlers/user_handler.go
type UpdateUserRequest struct {
    Name  *string `json:"name,omitempty"`
    Email *string `json:"email,omitempty"`
}

// `PATCH /users/{id}“
func UpdateUserHandler(w http.ResponseWriter, r *http.Request) {
    // URL から ID を取得
    idStr := strings.TrimPrefix(r.URL.Path, "/users/")
    if idStr == "" {
        respondWithError(w, http.StatusBadRequest, "IDが必要です")
        return
    }

    // ID を整数に変換
    id, err := strconv.Atoi(idStr)
    if err != nil {
        respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
        return
    }

    var req UpdateUserRequest
    defer r.Body.Close()

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondWithError(w, http.StatusBadRequest, "無効なリクエストボディ")
        return
    }

    if req.Name == nil && req.Email == nil {
        respondWithError(w, http.StatusBadRequest, "更新するフィールドを指定してください")
        return
    }

    user, err := models.UpdateUser(id, req.Name, req.Email)
    if err != nil {
        if err.Error() == "sql: no rows in result set" {
            respondWithError(w, http.StatusNotFound, "ユーザーが見つかりません")
        } else {
            respondWithError(w, http.StatusInternalServerError, "ユーザー更新中にエラーが発生しました")
        }
        return
    }

    respondWithJson(w, http.StatusOK, user)
}
```

### ルーティング
`UpdateUser` を `PATCH /users/{id}` に紐付けます。

```go:router/router.go
func SetupRoutes() *http.ServeMux {
    mux := http.NewServeMux()

    //

    mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            handlers.GetUserHandler(w, r)
        case http.MethodPatch:
            handlers.UpdateUserHandler(w, r)
        default:
            methodNotAllowedHandler(w)
        }
    })

    return mux
}
```

### 動作確認
正常系
```
$ curl -X  PATCH -i -d '{"name": "高田こうた", "email": "takada@example.com"}' localhost:8080/users/2
HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 04 Apr 2025 06:14:48 GMT
Content-Length: 106

{"id":2,"name":"高田こうた","email":"takada@example.com","created_at":"2025-03-30T23:10:58.343966Z"}
```

異常系
```
$ curl -X  PATCH -i -d '{}' localhost:8080/users/2
HTTP/1.1 400 Bad Request
Content-Type: application/json
Date: Fri, 04 Apr 2025 06:15:13 GMT
Content-Length: 67

{"error":"更新するフィールドを指定してください"}
```


## DELETE /users/{id} API
最後に、ユーザーを削除するAPI `DELETE /users/{id}` を実装します。

### データ操作
`Exec` を使って、実行結果だけが重要なSQLを発行しています。
`result` は `sql.Result` 型で、何行更新されたか・最後に挿入したIDなどの情報を持ちます。

`result.RowsAffected()`で何行が実際に変更されたかを確認しています。
該当IDの削除成功の場合、`rowsAffected == 1`となります。
該当IDが存在しない場合、`rowsAffected == 0`となるのでエラーを発生させます。

```go:models/user.go
func DeleteUser(id int) error {
    result, err := config.DB.Exec("DELETE FROM users WHERE id = $1", id)
    if err != nil {
        return fmt.Errorf("削除に失敗しました: %w", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("削除結果の確認に失敗しました: %w", err)
    }
    if rowsAffected == 0 {
        return fmt.Errorf("sql: no rows in result set")
    }

    return nil
}

```

### ハンドラー
DELETE リクエストを受け取る HTTP ハンドラー を作ります。

今回も`sql: no rows in result set`を受け取った場合は`404 Not Found`を返却しています。
成功時は、`204 No Content`のみを返却しています。

```go:handlers/user_handler.go
// `DELETE /users/{id}`
func DeleteUserHandler(w http.ResponseWriter, r *http.Request) {
    // URL から ID を取得
    idStr := strings.TrimPrefix(r.URL.Path, "/users/")
    if idStr == "" {
        respondWithError(w, http.StatusBadRequest, "IDが必要です")
        return
    }

    // ID を整数に変換
    id, err := strconv.Atoi(idStr)
    if err != nil {
        respondWithError(w, http.StatusBadRequest, "IDは数値である必要があります")
        return
    }

    err = models.DeleteUser(id)
    if err != nil {
        if err.Error() == "sql: no rows in result set" {
            respondWithError(w, http.StatusNotFound, "ユーザーが見つかりません")
        } else {
            respondWithError(w, http.StatusInternalServerError, "ユーザー削除中にエラーが発生しました")
        }
        return
    }

    w.WriteHeader(http.StatusNoContent)
}
```

### ルーティング
`DeleteUser` を `DELETE /users/{id}` に紐付けます。

```go:router/router.go
func SetupRoutes() *http.ServeMux {
    mux := http.NewServeMux()

    //

    mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodPatch:
            handlers.UpdateUserHandler(w, r)
        case http.MethodDelete:
            handlers.DeleteUserHandler(w, r)
        default:
            methodNotAllowedHandler(w)
        }
    })

    return mux
}
```

### 動作確認
正常系
```
$ curl -X DELETE -i localhost:8080/users/7
HTTP/1.1 204 No Content
Date: Sat, 05 Apr 2025 23:53:28 GMT
```

異常系
```
$ curl -X DELETE -i localhost:8080/users/99
HTTP/1.1 404 Not Found
Content-Type: application/json
Date: Sat, 05 Apr 2025 23:54:23 GMT
Content-Length: 49

{"error":"ユーザーが見つかりません"}
```