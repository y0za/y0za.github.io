---
layout: post
title: "vimrc読書会でライフゲーム"
description: ""
date: 2017-02-22
tags: ["vim"]
comments: true
share: true
---

![](/assets/images/reading-vimrc-life/main.png)
[y0za/reading-vimrc-life](https://github.com/y0za/reading-vimrc-life)  

vimrc読書会のユーザーごとの参加した回の表がライフゲームっぽかったので、実際にライフゲーム化させてみた。  
([詳しくはgitterのログ](https://gitter.im/vim-jp/reading-vimrc?at=5839a7ff381827c24d832fb3)を参照)

## 実現方法
- Chromeの拡張機能
- MutationObserverで表の生成を監視
- 対象の部分をライフゲーム化させたものをVueで生成して置き換え

Angularで生成している部分にVueをねじ込んでいるので良い子は真似しないように...  
試したい人はcloneして`yarn run build`して生成されたdistディレクトリをChromeの拡張機能として読み込んでみてください。

## 実行例
自分の表だと参加回数が少なくてパッとしない結果だったので他の人ので実行させてみた結果を載せておきます。

### mattnさんの場合
![](/assets/images/reading-vimrc-life/play-mattn.gif)
ちょうどいい感じの穴空き具合  
mattnさんレベルのVimmerにもなると、4年以上も前からこうなることを予見して穴を空けておいてくれたのかもしれない...

### thincaさんの場合
![](/assets/images/reading-vimrc-life/play-thinca.gif)
どうやらライフゲームの世界でもマンボウの生存率は極めて低いようだ (これが言いたかっただけ)
