---
title: "docker-compose で Go + PostgreSQL の環境構築をする"
emoji: "🐳 "
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "postgresql", "docker"]
published: true
---
## はじめに
docker-compose で Go + PostgreSQL の環境構築をしてみました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-postgres-docker

## ディレクトリ構成
```
myapp
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── main.go
```

## アプリの作成
検証のための簡単なアプリを作成します。

`go mod init` を実行します。
```:go.mod
module github.com/t-shimpo/go-postgres-docker

go 1.24.0
```

`/`にアクセスすると `Hello, World!` と表示されるプログラムを実装しました。
```go:main.go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server running on port 8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
`go run main.go`を実行し、`localhost:8080`にアクセスすると `Hello World!` と表示されます。

## Goコンテナを作成
続いて、GoをDockerで動かせるようにします。
以下の`Dockerfile`と`docker-compose.yml`を追加しました。


```Dockerfile:Dockerfile
FROM golang:1.24

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["go", "run", "main.go"]
```

`COPY go.mod go.sum ./`について、後の実装で`go.sum`が更新されるので`go.sum`も含めています。
ファイルがない場合エラーになるので、空ファイルで`go.sum`を追加しておきます。

```yml:docker-compose.yml
services:
  app:
    build: .
    container_name: app
    ports:
      - "8080:8080"
    volumes:
      - .:/app
```

以下のコマンドを実行して、アプリが実行できるか確認します。
```
$ docker-compose up -d
```
`localhost:8080`にアクセスし `Hello World!` と表示されることを確認します。

## PostgreSQLコンテナを作成
続いて、PostgreSQLをDockerで動かせるようにします。
`docker-compose.yml`を更新しました。

```yml:docker-compose.yml
services:
  app:
    build: .
    container_name: app
    ports:
      - "8080:8080"
    volumes:
      - .:/app
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://user:password@db:5432/mydb?sslmode=disable

  db:
    image: postgres
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```
今回は簡単に、DB接続用の情報をGoアプリで`DATABASE_URL`として取得できるようにしています。

dockerを起動して、PostgreSQLのコンテナに入れるか確認します。
```
$ docker-compose up -d
```
以下のコマンドを実行して、PostgreSQLにログインできるか確認します。
※`postgres`はコンテナ名です。
```
$ docker exec -it postgres psql -U user -d mydb
```
成功すれば`mydb=#` のようなプロンプトが表示されます。 

#### 補足
以下の設定でdbコンテナが起動してからappコンテナを起動するようにしています。
```yml
depends_on:
  - db
```
ただし、PostgreSQL の起動完了を保証するわけではないため、アプリ側でリトライ処理を入れるのが一般的です。


以下の設定はデータの永続に関する設定です。
```yml
volumes:
  postgres-data:
```
`postgres-data` というボリュームを作成し、PostgreSQL のデータ(`/var/lib/postgresql/data`)を永続化しています。
これにより、コンテナを削除してもデータが保持されます。

## 接続確認
GoアプリとpostgresDBの接続を確認します。
`main.go`を以下のように更新しました。

```go:main.go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "net/http"
    "os"

    _ "github.com/lib/pq"
)

func handler(w http.ResponseWriter, r *http.Request) {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        http.Error(w, "DATABASE_URL not set", http.StatusInternalServerError)
        return
    }

    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    err = db.Ping()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer db.Close()

    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Database connection successful"))
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server running on port 8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
`github.com/lib/pq`はPostgreSQL用のドライバです。
PostgreSQLに接続するにはこちらが必要となります。
外部パッケージを追加したので、`go mod tidy`も実行しておきます。

dockerを起動して動作確認します。
```
$ docker-compose up -d
```
`localhost:8080`にアクセスし `Database connection successful` と表示されれば成功です。

## 参考資料
https://hub.docker.com/_/golang

https://hub.docker.com/_/postgres

https://docs.docker.jp/compose/toc.html