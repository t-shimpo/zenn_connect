---
title: "【Go】Echo + GORM(v2) + MySQLでREST API（CRUD）を作成する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "echo", "gorm", "mysql", "docker"]
published: true
---

## はじめに
Go + Echo + GORM(v2) + MySQL で簡単なREST APIを作成しました。

なお、作成したものは、以下のレポジトリで公開しています。
https://github.com/t-shimpo/go-echo-gorm-rest

### Echo概要
EchoはGoのWebフレームワークです。
軽量・高パフォーマンス・拡張性が高いといった特徴があります。

https://echo.labstack.com/

### GORM概要
GORMはGoのORMの1つです。
Githubのスター数も多く、人気のライブラリです。
機能が充実していて、マイグレーションも可能です。

https://gorm.io/

[日本語のドキュメント](https://gorm.io/ja_JP/docs/index.html)もあります。

## ディレクトリ構成
```
myapp
├── controller
│    └── user.go
├── model
│    ├── db.go
│    └── user.go
├── api.dockerfile
├── db.dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
└── main.go
```

## Dockerで環境構築をする
まず、DockerでGo + MySQLの環境構築をします。
なお、以下の記事でも、同じように実装していますので、参考にしていただければと思います。
https://zenn.dev/shimpo/articles/setup-go-mysql-with-docker-compose

`go mod init`を実行します。
```:go.mod
module github.com/t-shimpo/go-echo-gorm-rest

go 1.19
```

Goコンテナ用のdockerfileを作成します。
```dockerfile:app.dockerfile
FROM golang:1.19

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download && go mod verify

CMD ["go", "run", "main.go"]
```
MySQLコンテナ用のdockerfileを作成します。
```dockerfile:db.dockerfile
FROM mysql:8.0
ENV LANG ja_JP.UTF-8
```

docker-compose.ymlを作成します。
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

## Echoでサーバーを立てる
Echoをインストールします。
```
$ go get github.com/labstack/echo/v4
```

サーバーを立ち上げて**Hello World**と表示してみます。
```go:main.go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":8080"))
}
```

**main.go**を追加したら、`docker-compose up`で実行します。
`http://localhost:8080/`にアクセスするか、
`curl localhost:8080`を実行し、**Hello World**と表示されることを確認します。

## GORMを導入する
### DBと接続する
GORMとMySQL Driverをインストールします。
```
$ go get gorm.io/gorm
$ go get gorm.io/driver/mysql
```

GORMを使ってDBに接続するために、以下のファイルを作成しました。
(参考：https://gorm.io/ja_JP/docs/connecting_to_the_database.html)
※`dsn`の部分はDBの設定に応じて書き換えてください。
```go:model/db.go
package model

import (
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var DB *gorm.DB
var err error

func init() {
	dsn := "tester:password@tcp(db:3306)/test?charset=utf8mb4&parseTime=True&loc=Local"
	DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalln(dsn + "database can't connect")
	}
}
```

**main.go**を変更し、[Ping](https://pkg.go.dev/database/sql#DB.Ping)でDBとの接続を確認してみます。
```go:main.go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/t-shimpo/go-echo-gorm-rest/model"
)

func connect(c echo.Context) error {
	db, _ := model.DB.DB()
	defer db.Close()
	err := db.Ping()
	if err != nil {
		return c.String(http.StatusInternalServerError, "DB接続失敗しました")
	} else {
		return c.String(http.StatusOK, "DB接続しました")
	}
}

func main() {
	e := echo.New()
	e.GET("/", connect)
	e.Logger.Fatal(e.Start(":8080"))
}
```
`http://localhost:8080/`にアクセスするか、
`curl localhost:8080`を実行し、接続を確認します。


## CRUDを実装する
### Userテーブルの作成
今回は、usersテーブルを作成し、Create, Read, Update, Deleteのそれぞれを実装していきます。

まず、以下のファイルでUser構造体を定義します。
```go:model/user.go
package model

import "time"

type User struct {
	ID        uint      `json:"id  param:"id""`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}
```
なお、GORMはIDフィールドをデフォルトでPRIMARY KEYとして扱います。
（参考：https://gorm.io/ja_JP/docs/conventions.html#%E4%B8%BB%E3%82%AD%E3%83%BC%E3%81%A8%E3%81%97%E3%81%A6%E3%81%AE-ID）

そして、**model/db.go**を改善します。
```diff go:model/db.go
func init() {
	dsn := "tester:password@tcp(db:3306)/test?charset=utf8mb4&parseTime=True&loc=Local"
	DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalln(dsn + "database can't connect")
	}
++	DB.AutoMigrate(&User{})
}
```
テーブルの作成するために、GORMの[AutoMigrate](https://pkg.go.dev/gorm.io/gorm#DB.AutoMigrate)メソッドを使いました。
引数に構造体を渡すことで、テーブルを作成できます。
（参考：https://gorm.io/ja_JP/docs/migration.html）

### Create
**controller/user.go**を作成し、ユーザー作成処理を実装します。
```go:controller/user.go
package controller

import (
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/t-shimpo/go-echo-gorm-rest/model"
)

func CreateUser(c echo.Context) error {
	user := model.User{}
	if err := c.Bind(&user); err != nil {
		return err
	}
	model.DB.Create(&user)
	return c.JSON(http.StatusCreated, user)
}
```
`Bind`メソッドで、リクエストボディからデータを取得できます。
（参考：https://echo.labstack.com/guide/binding/#struct-tag-binding）

また、`DB.Create`の引数に構造体のポインタを渡すことで、レコードを作成できます。
CreatedAtと、UpdatedAtは、値がゼロ値であれば、現在時刻が入ります。
（参考：https://gorm.io/ja_JP/docs/conventions.html#%E3%82%BF%E3%82%A4%E3%83%A0%E3%82%B9%E3%82%BF%E3%83%B3%E3%83%97%E3%81%AE%E3%83%88%E3%83%A9%E3%83%83%E3%82%AD%E3%83%B3%E3%82%B0）


作成したCreateUser関数を実行できるように、**main.go**を以下のようにします。
```go:main.go
package main

import (
	"github.com/labstack/echo/v4"
	"github.com/t-shimpo/go-echo-gorm-rest/controller"
	"github.com/t-shimpo/go-echo-gorm-rest/model"
)

func main() {
	e := echo.New()
	db, _ := model.DB.DB()
	defer db.Close()

	e.POST("/users", controller.CreateUser)
	e.Logger.Fatal(e.Start(":8080"))
}
```

curlで作成してみます。
```
$ curl -X POST -H "Content-Type: application/json" -d '{"name":"テスト太郎"}' localhost:8080/users
{"id":1,"name":"テスト太郎","created_at":"2022-12-30T08:41:51.838Z","updated_at":"2022-12-30T08:41:51.838Z"}
```
データベースも確認してみます。
```
mysql> SELECT * FROM users;
+----+-----------------+-------------------------+-------------------------+
| id | name            | created_at              | updated_at              |
+----+-----------------+-------------------------+-------------------------+
|  1 | テスト太郎      | 2022-12-30 08:41:51.838 | 2022-12-30 08:41:51.838 |
+----+-----------------+-------------------------+-------------------------+
1 row in set (0.00 sec)
```

### Read
続いて、ユーザー取得処理を追加します。
**controller/user.go**で、全件取得する処理と1件取得する処理を以下のように実装しました。

```go:controller/user.go
func GetUsers(c echo.Context) error {
	users := []model.User{}
	model.DB.Find(&users)
	return c.JSON(http.StatusOK, users)
}

func GetUser(c echo.Context) error {
	user := model.User{}
	if err := c.Bind(&user); err != nil {
		return err
	}
	model.DB.Take(&user)
	return c.JSON(http.StatusOK, user)
}
```
#### 全件取得
FindメソッドにUser構造体のポインタを渡すことで、全件取得できます。

#### 1件取得
User構造体で、IDに`param`のタグをつけましたが、それによって`Bind`メソッドでパスパラメータを取得できます。
そしてTakeメソッドに渡すことで、`/users/1`のような形で、指定したIDのユーザーを1件取得できます。


作成したCreateUser関数を実行できるように、**main.go**を以下のようにします。
```go:main.go
func main() {
	//...
	e.GET("/users", controller.GetUsers)    // 追加
	e.GET("/users/:id", controller.GetUser) // 追加
	e.POST("/users", controller.CreateUser)
	e.Logger.Fatal(e.Start(":8080"))
}
```

curlで取得してみます。
```
$ curl -X GET localhost:8080/users
[{"id":1,"name":"テスト太郎","created_at":"2022-12-30T08:41:51.838Z","updated_at":"2022-12-30T08:41:51.838Z"},{"id":2,"name":"テスト花子","created_at":"2022-12-30T08:52:39.809Z","updated_at":"2022-12-30T08:52:39.809Z"}]
```
```
$ curl -X GET localhost:8080/users/1
{"id":1,"name":"テスト太郎","created_at":"2022-12-30T08:41:51.838Z","updated_at":"2022-12-30T08:41:51.838Z"}
```

### Update
Updateは以下のように実装できます。
```go:controller/user.go
func UpdateUser(c echo.Context) error {
	user := model.User{}
	if err := c.Bind(&user); err != nil {
		return err
	}
	model.DB.Save(&user)
	return c.JSON(http.StatusOK, user)
}
```

```go:main.go
func main() {
	//...
	e.GET("/users", controller.GetUsers)
	e.GET("/users/:id", controller.GetUser)
	e.POST("/users", controller.CreateUser)
	e.PUT("/users/:id", controller.UpdateUser) // 追加
	e.Logger.Fatal(e.Start(":8080"))
}
```

curlで更新してみます。
```
$ curl -X PUT -H "Content-Type: application/json" -d '{"name":"山田たろう"}' localhost:8080/users/1
{"id":1,"name":"山田たろう","created_at":"0001-01-01T00:00:00Z","updated_at":"2022-12-30T09:20:15.99Z"}
```

### Delete

Deleteは以下のように実装できます。
```go:controller/user.go
func UpdateUser(c echo.Context) error {
	user := model.User{}
	if err := c.Bind(&user); err != nil {
		return err
	}
	model.DB.Save(&user)
	return c.JSON(http.StatusOK, user)
}
```

```go:main.go
func main() {
	//...
	e.GET("/users", controller.GetUsers)
	e.GET("/users/:id", controller.GetUser)
	e.POST("/users", controller.CreateUser)
	e.PUT("/users/:id", controller.UpdateUser)
	e.DELETE("/users/:id", controller.DeleteUser) // 追加
	e.Logger.Fatal(e.Start(":8080"))
}
```

curlで削除してみます。
```
$ curl -X DELETE localhost:8080/users/1
{"id":1,"name":"","created_at":"0001-01-01T00:00:00Z","updated_at":"0001-01-01T00:00:00Z"}
```

## 参考資料
https://echo.labstack.com/
https://gorm.io/
https://gorm.io/ja_JP/docs/index.html
