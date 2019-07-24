---
layout: post
title: "GAE Go 1.11への移行作業"
description: ""
date: 2019-07-24
tags: ["App Engine", "Go"]
comments: true
share: true
---

[2019-10-01からApp EngineにGo 1.9以下のバージョンのアプリがデプロイできなくなる](https://cloud.google.com/appengine/docs/deprecations/)との通達を受けて、Go 1.11への移行作業を行ったのでざっくりとまとめておきます。  
最終的にはApp Engine特有のAPIへの依存のない状態を目指す予定ですが、今回は初期段階として`appengine.Main()`を使用したものとなります。


## 作業内容


### main.goの変更
- `package main`に変更
- init関数をmain関数に置き換え
- `appengine.Main()`を追加

[公式のサンプル](https://github.com/GoogleCloudPlatform/golang-samples/blob/master/appengine/go11x/helloworld/helloworld.go)をベースに改変したコード
```go
package main

import (
	"fmt"
	"net/http"

	"google.golang.org/appengine"
)

func main() {
	http.HandleFunc("/", indexHandler)

	appengine.Main()
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path != "/" {
		http.NotFound(w, r)
		return
	}
	fmt.Fprint(w, "Hello, World!")
}
```


### app.yamlの変更
- `runtime: go111`に変更
- `api_version: go1.9`を除去
- handlersの`script: _go_app`を`script: auto`に置き換え
- includesの除去
- handlersのloginの除去 (オプション)

上記以外にも[Deprecatedになった設定](https://cloud.google.com/appengine/docs/standard/go111/go-differences#app-yaml)がいくつかあるので注意

#### includesの除去
自分の場合、Gitで管理しないように環境変数(env_variables)を定義するファイルを個別に作ってincludesで読み込んでいました。
この用途なら[github.com/joho/godotenv](https://github.com/joho/godotenv)で代替できますし、それ以外の用途でも単純にファイル結合(e.g. `cat app_base.yaml hoge.yaml > app.yaml`)させるなどで一応対応できるかと思います。

#### handlersのloginの除去
Go 1.11ではloginはまだサポートされているのでそのままでも問題ありませんが、将来的に使えなくなるので一応触れておきます。  
TaskQueueのハンドラで`login: admin`を使用してリクエストの検証をしていた場合には、[HTTPヘッダー](https://cloud.google.com/appengine/docs/standard/go/taskqueue/push/creating-handlers?hl=ja#reading_request_headers)の`X-Appengine-Taskname`などが空ではないことを確認すればいいかと思わます。
[公式のサンプル](https://github.com/GoogleCloudPlatform/golang-samples/blob/master/appengine/go11x/tasks/handle_task/handle_task.go#L56-L63)  
それ以外で使用している場合には[何かしらの認証手段](https://cloud.google.com/appengine/docs/standard/go111/authenticating-users)を導入する必要があるとのこと。


### go.modの利用
vendorを利用している場合などはそのままでも問題ないかと思いますがgo.modが使えるようになったので移行します。(自分はdepで管理していました)
- デプロイの際のビルドとなるべく環境を合わせるため[goenv](https://github.com/syndbg/goenv)などを利用してGo 1.11環境にする
- go.modを作成 `go mod init`
- module名やコード内でのimportのパスは適切なものに調整
- もし[google.golang.org/appengine](https://godoc.org/google.golang.org/appengine)が古い場合はアップデート `go get -u google.golang.org/appengine`
- 一度ビルドしてgo.sumの作成などを行う `go build .`
- 不要になったvendorディレクトリなどは削除しておく


### gcloudコマンドでデプロイ
vendorディレクトリの関係でappcfg.pyを用いてデプロイしていましたが、Go 1.11はサポートされていないのでgcloudコマンドでデプロイするようにします。
```
$ gcloud app deploy app.yaml index.yaml queue.yaml
```
サービスアカウントでデプロイする場合などに必要な権限は[こちら](https://cloud.google.com/appengine/docs/standard/go111/granting-project-access)を参照


## 参考
- [Migrating your App Engine app from Go 1.9 to Go 1.11](https://cloud.google.com/appengine/docs/standard/go111/go-differences)
- [App Engine Standard Go 1.9 migration to Go 1.11に最低限必要なこと](https://github.com/gcpug/nouhau/tree/master/app-engine/note/gaego19-migration-gaego111)
- [GAE/Go1.9から1.11にマイグレーションしようとして躓いたところ](https://techblog.ap-com.co.jp/entry/2019/04/19/174903)
