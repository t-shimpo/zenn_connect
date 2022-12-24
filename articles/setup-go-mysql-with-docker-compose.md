---
title: "docker-composeでGo + MySQLの環境構築をする"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "mysql", "docker"]
published: false
---
docker-composeでGo + MySQLの環境構築をしてみました。

なお、作成したものは以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-mysql-docker

## 環境
- macOS Monterey 12.4
- Docker 20.10.21

## ディレクトリ構成
```
myapp
├── database
│    └── migration.sql
├── api.dockerfile
├── db.dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
└── main.go
```

## アプリの作成
検証のため、簡単なアプリを作成します。

```:go.mod
module github.com/t-shimpo/go-mysql-docker

go 1.19
```

```go:main.go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World")
}

func main() {
	http.HandleFunc("/", handler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
`go run main.go`を実行し、`localhost:8080`にアクセスすると、**Hello World**と表示されます。

## Goコンテナを作成
### Dockerfileの作成
Dockerイメージを作成するための`Dockerfile`を作成します。
docker hubの公式イメージ（[こちらのページ](https://hub.docker.com/_/golang)）を参考にしました。

```dockerfile:app.dockerfile
FROM golang:1.19

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download && go mod verify

CMD ["go", "run", "main.go"]
```
ワークディレクトリを設定し、
`go mod download`で**go.mod**に記載されたモジュールをダウンロード、
`go run main.go`で実行しています。

### docker-compose.ymlを作成
Docker Composeでアプリを実行できるようにします。
以下のとおり`docker-compose.yml`を作成しました。

```yml:docker-compose.yml
version: '3.8'

services:
  app:
    container_name: app
    build:
      context: .
      dockerfile: app.dockerfile
    tty: true
    ports:
      - 8080:8080
    volumes:
      - .:/app
```
### 実行する
以下のコマンドでDocker Composeを起動します。

```
docker-compose up -d
```
`http://localhost:8080`にアクセスして、アプリが立ち上がっていることを確認します。

## MySQLコンテナを作成
### SQLファイルの作成
今回は、検証のために、MySQLからデータ取得したデータを表示させたいと思います。
そのために、データを挿入する以下のSQLファイルを作成しました。

```sql:database/migration.sql
CREATE TABLE `users`
(
  id bigint auto_increment,
  name varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
);

INSERT INTO users (`name`) VALUES ('山田太郎'), ('鈴木花子');
```
usersテーブルを追加し、データをinsertしています。

### Dockerfileの作成
MySQLコンテナ用のDockerfileは以下のようにしました。

```dockerfile:db.dockerfile
FROM mysql:8.0
ENV LANG ja_JP.UTF-8

COPY ./database/*.sql /docker-entrypoint-initdb.d/
```
先ほど作成したSQLファイルを実行してデータを挿入するために、
`COPY ./database/*.sql /docker-entrypoint-initdb.d/`を記述しました。
`/docker-entrypoint-initdb.d`にマウントしたディレクトリに「.sql」「.sh」「.sql.gz」という拡張子でファイルを配置しておくと、コンテナを生成・起動時に、それらのファイルが実行されます。
（※docker hubの[こちらのページ](https://hub.docker.com/_/mysql/#docker-secrets)の**Initializing a fresh instance**に記載があります。）

### DBからデータを取得する処理をアプリに追加
**main.go**は最終的に以下のようにしました。

```go:main.go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"log"
	"net/http"

	_ "github.com/go-sql-driver/mysql"
)

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World")
}

func getUsers() []*User {
	db, err := sql.Open("mysql", "tester:password@tcp(db:3306)/test")
	if err != nil {
		panic(err)
	}
	defer db.Close()

	results, err := db.Query("SELECT * FROM users")
	if err != nil {
		panic(err)
	}

	var users []*User
	for results.Next() {
		var u User
		err := results.Scan(&u.ID, &u.Name)
		if err != nil {
			panic(err)
		}
		users = append(users, &u)
	}
	return users
}

func usersPage(w http.ResponseWriter, r *http.Request) {
	users := getUsers()
	json.NewEncoder(w).Encode(users)
}

func main() {
	http.HandleFunc("/", handler)
	http.HandleFunc("/users", usersPage)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
getUsers関数がDBからuserのリストを取得して返却しており、
`/users`のパスにアクセスすると、json形式でuserのリストを確認できます。

### docker-compose.ymlを更新
**docker-compose**は以下のように更新しました。

```yml:docker-compose.yml
version: '3.8'

services:
  app:
    container_name: app
    build:
      context: .
      dockerfile: app.dockerfile
    tty: true
    ports:
      - 8080:8080
    depends_on:
      - db
    volumes:
      - .:/app

  db:
    container_name: db
    build:
      context: .
      dockerfile: db.dockerfile
    tty: true
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: "test"
      MYSQL_USER: "tester"
      MYSQL_PASSWORD: "password"
    volumes:
      - type: volume
        source: mysql_data
        target: /var/lib/mysql
    networks:
      - default

networks:
  default:
volumes:
  mysql_data:
```

### 実行する
Docker Composeを起動します。
```
docker-compose up -d
```
`localhost:8080/users`にアクセスすると以下が表示されました。

![](/images/setup-go-mysql-with-docker-compose/users.png)

## 参考資料
https://hub.docker.com/_/golang

https://hub.docker.com/_/mysql/#docker-secrets
