---
layout: post
title: "GAEへのqueue.yamlのデプロイに失敗する"
description: ""
date: 2019-07-25
tags: ["App Engine"]
comments: true
share: true
---

前回の[GAE Go 1.11への移行作業](/2019-07-24/gae-go111-migration/)の続き。  

## 内容
CIからサービスアカウントで`gcloud app deploy app.yaml index.yaml queue.yaml`といったコマンド実行した際に以下のようなエラーが発生しました。
```
Services to deploy:

descriptor:      [/root/project/app.yaml]
source:          [/root/project]
target project:  [xxxxxxxx]
target service:  [default]
target version:  [2019xxxxxxxxxxx]
target url:      [https://xxxxxxxx.appspot.com]


Configurations to update:

descriptor:      [/root/project/index.yaml]
type:            [datastore indexes]
target project:  [xxxxxxxx]


descriptor:      [/root/project/queue.yaml]
type:            [task queues]
target project:  [xxxxxxxx]


ERROR: (gcloud.app.deploy) PERMISSION_DENIED: The caller does not have permission
Exited with code 1
```
エラーメッセージから何らかの権限がないと見て取れます。  

このままでは原因がわからないので`--log-http --verbosity=debug`をつけて実行してみると以下のようなログが出力されました。
```
==== request start ====
uri: https://serviceusage.googleapis.com/v1/projects/xxxxxxxx/services/cloudtasks.googleapis.com?alt=json
method: GET
== headers start ==
Authorization: --- Token Redacted ---
accept: application/json

-- 中略 -- 

-- body start --
{
  "error": {
    "code": 403,
    "message": "The caller does not have permission",
    "status": "PERMISSION_DENIED"
  }
}

-- body end --
```

どうやらsserviceusageというAPIへのGETリクエストに対して権限がないから失敗している様子。  
ググってみると同様の問題がIssue Trackerに報告されていました。  
[Permission denied when deploying queue.yaml](https://issuetracker.google.com/issues/137078982)  
このIssueに対するコメントでroles/serviceusage.serviceUsageViewerの権限を追加したら解決したとあったので、"Service Usage 閲覧者"の権限をサービスアカウントに追加したら無事デプロイできるようになりました。



## まとめ
`gcloud app deploy`でqueue.yamlをデプロイする際には"Service Usage 閲覧者"の権限が必要
