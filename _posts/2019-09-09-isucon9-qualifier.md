---
layout: post
title: "ISUCON9 予選惜敗"
description: ""
date: 2019-09-09
tags: ["ISUCON", "Go"]
comments: true
share: true
---

[yuki2006](https://twitter.com/yuki2006_kd)さんと2人で参加して最終スコア9560ｲｽｺｲﾝでした。  
使用した言語はGo

## 個人的攻略ポイント
Nginxのアクセスログの集計結果から`/users/transactions.json`に時間がかかっていたことがわかったのでそこを改善する必要があると判断。実行しているSQLも重そうだが、取得したitemsをforループで回している中で配送サービスAPIにリクエストを投げているのがヤバそう。  

とりあえず並行にリクエストを投げても問題なさそうだったのでsync.WaitGroupとgoroutineを使って並行化。スコアアップするもののまだ重い。  

そもそも配送サービスAPIに問い合わせる必要があるのかよく考えてみる。APIからのレスポンスで使用しているのはstatusのプロパティでこれは配送サービス側が保持している配送状態を表す。
配送状態がどのように変化するかはアプリケーション仕様書に書いてある。

以下は仕様書から ISUCARI ステータス遷移表 を抜粋したもの

|                        | WHO    | items    | transaction_evidences | shippings           |
|------------------------|:------:|:--------:|:---------------------:|:-------------------:|
| /sell （出品）         | 出品者 | on_sale  | -                     | -                   |
| /buy  （購入）         | 購入者 | trading  | wait_shipping         | initial             |
| /ship （集荷予約）     | 出品者 |   ↓      |   ↓                   | wait_pickup         |
| /ship_done （発送完了）| 出品者 |   ↓      | wait_done             | shipping or done    |
| /complete （取引完了） | 購入者 | sold_out | done                  | done                |

この表からいくつかわかることがある。
- 購入者が`/buy`を叩いて完了した際に配送状態は`initial`となる
- 出品者が`/ship`を叩いて完了した際に配送状態は`wait_pickup`となる
- 出品者が`/ship_done`を叩いて完了した後の配送状態は`shipping`もしくは`done`のどちらかである
- 購入者が`/complete`を叩いて完了した際に配送状態は`done`で確定している

つまり`/ship_done`以外の場合には配送状態は一意に特定できる。そして、それと対応するように各エンドポイントの処理で配送状態をshippingsテーブルのstatusに保存している。

以上のことから、shippingsテーブルのstatusが`shipping`である場合のみそれが配送中なのか配送済みなのかは配送サービスAPIに問い合わせてみないとわからないのであって、それ以外の場合はDBに保存されている値を使えばいいことになる。

実際にAPIアクセスを削って、ついでに問い合わせた結果が`done`だった場合にDBの値を更新してみたところ、3700だったスコアが9000くらいに伸びました。


## 使ったツールなど
- rsync: ファイル転送
- [tkuchiki/alp](https://github.com/tkuchiki/alp): Nginxのアクセスログのプロファイラ
- mysqldumpslow: MySQLのスローログ解析
- [catatsuy/notify_slack](https://github.com/catatsuy/notify_slack): サーバ上での集計結果などをスラックに通知
- Go pprofのWebUI: GoのCPUボトルネック解析
- [najeira/measure](https://github.com/najeira/measure): Goの処理時間計測

手元のMacのGo 1.13でクロスコンパイルしてrsyncでサーバに実行ファイルを送るといった手段をとっていまして、もしサーバ上のGoのバージョンが古かったりしても気にする必要がなかったので割とよかった気がします。  
また、今回の事前練習中にnotify_slackのことを知り使ってみたところ、journalctlでのログをそのままSlackに流し込んだりalpやmysqldumpslowでの集計結果をSlack上で共有するのが非常に簡単でとても便利でした。

## 感想
個人的に上記のISUCARI ステータス遷移表は運営側から与えられた大きなヒントだと思っていて、仕様をちゃんと理解すると不要な処理だとわかるといった問題がよくできているな、と感心しました。  
ログイン処理や購入処理などもボトルネックになっていたのはわかっていましたが、ツールの使用方法を間違えて時間を消費したり`/users/transactions.json`に時間がかかってしまったりで他の部分の改善案を考える時間が足りませんでした。予選突破まであと少しと悔いが残る結果になってしまったので、来年リベンジできるよう精進しようと思います。
