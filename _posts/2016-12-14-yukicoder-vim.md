---
layout: post
title: "yukicoderにVim scriptをぶち込んだ話"
description: ""
date: 2016-12-14
tags: ["vim"]
comments: true
share: true
---


この記事は [Vim (その2) Advent Calendar 2016](http://qiita.com/advent-calendar/2016/vim2) の14日目の記事です。

## はじめに
VimConf 2016が終わっても冷めないVim熱... 最近は、更なるVim力のステップアップを求めて何かプラグインを作ってみたいなー、と思うようになってきました。Vimのプラグインを作る上で避けては通れないのがVim scriptという言語で、それを効率よく学びたいなー、と思ったのが事の発端です。


## yukicoderとは
[https://yukicoder.me/](https://yukicoder.me/)から引用  

> 競技プログラミングのスキルの練習として、回答したい人と出題したい人をつなぐサービスです。
> 競技プログラミングは、出題することもとてもアルゴリズムの勉強になります。
> 完璧に出題できなくても、気軽にお互い指摘しあって向上していこうという方針で行っております。
> 競技プログラミングで遊ぼう！という感じで勉強会などで使って便利なサービスを目指しております。

競技プログラミングの練習や出題ができるサービスですね。競技プログラミングと言われると難しそうなイメージがありますが、yukicoderでは問題ごとに難易度が星の数で表されていて星１つの問題だったらアルゴリズムを使うというより、プログラミングのちょっとした実装力を問う問題が多いです。なので、新しいプログラミング言語の文法を覚えたいときに簡単な問題を幾つか解いてみる、というのは実際に手を動かすことになり学習手段として個人的におすすめです。  
yukicoderでは[様々なプログラミング言語](http://yukicoder.me/help/environments)がサポートされていて、そこにVim scriptを加えて効率的に学習できる環境を作ることが今回の目的となります。


## 本題
競技プログラミングでは幾つか解答の形式がありますが、yukicoderでは標準入力からパラメータを受け取ってそれを元に標準出力に結果を出力する、という動作のプログラムのソースコードを提出する形式となっています。つまり、yukicoderで使える言語の条件として標準入力と標準出力を扱える必要があります。...標準入力と標準出力を扱えるなんて大抵の言語だったらできて当たり前ですが、Vim scriptではどうすればいいのでしょうか？自分の知識ではさっぱりわからないのでググってみるといくつか参考になる記事が見つかりました。

- [シェル上で普通の言語のようにVim scriptを実行したい](http://koturn.hatenablog.com/entry/2015/07/25/043026)
- [vimをパイプにする](http://auewe.hatenablog.com/entry/2016/12/03/001000)
- [Vim script で AtCoder に参戦する方法](http://thinca.hatenablog.com/entry/20151003/1443853833)

1つ目はkoturnさんの記事で、`echo`などでの出力が標準エラー出力に出力されるのを標準出力にリダイレクトさせています。標準出力は扱うことができますが、自分の知識ではこれに標準入力を与える方法がわからなかったので断念。  
2つ目はまさかの同じAdvent Calendar 3日目のaueweさんの記事。入力をパイプで受け取って`-e`のオプションでコマンドを実行しています。標準入力も標準出力も扱えて良いのですが、パイプでの処理を前提としているのでこのままだとyukicoderで使うには少し扱いづらいです。  
3つ目はthincaさんの記事。AtCoderというyukicoderとは別の競技プログラミングのコンテストサイト向けにVim scriptを実行するための方法が書かれていて、まさに知りたかったことが載っていました。今回はこの記事を参考にして(パクって)起動用のコマンド等を用意することに...  

実際にできた起動コマンドがこちら  

```sh
vim -u NONE -i NONE -X -N -n -e -s -S _filename_ /dev/stdin -c qa!
```

各起動オプションの説明

- `-X` X serverに接続しない
- `-N` nocompatibleに設定
- `-u NONE` 設定ファイルや環境変数による初期化を行わない
- `-i NONE` viminfoの読み書きを行わない
- `-n` スワップファイルを使用しない
- `-e -s` サイレント(バッチ)モードとして起動 printやlistなどのコマンドの出力先が標準出力となる
- `-S {file}` 最初のファイルが読み込まれたあとに`{file}`を実行する
- `-c qa!` `qall!`を実行してVimを終了させる

はい、thincaさんのやつほぼそのまんまです(^_^;) `-X`は一応付けてみたけど必要ないかも... 実行するVim script側では、入力用のバッファを`getline(1, '$')`で内容を受け取って、出力するときは新たに出力用のバッファを作って結果を吐き出して`2,$print`等で標準出力に出力できます。  

このコマンドとVimをビルドするDockerfileを運営のyuki2006さんに投げつけて、めでたくyukicoderで使用できる言語として採用されました。
<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">【お知らせ】<br>・OLEの新設をしました。<br>・通常ジャッジ（小数誤差も含む）の速度が少々早くなりました。<br>・（要望から）「Vim script」言語が追加されました。<br>・質問にて、公開時に通知が飛んでなかったバグ（エンバグしていた）を修正しました。<a href="https://twitter.com/hashtag/yukicoder?src=hash">#yukicoder</a></p>&mdash; yukicoderお知らせアカウント (@yukicoder) <a href="https://twitter.com/yukicoder/status/807477740177756160">2016年12月10日</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


## 実際に解いてみた
試しに[簡単そうな問題を20問ほど解いた](https://yukicoder.me/users/27/submissions?lang_id=vim&status=AC&date_asc=enabled)ので、その中からいくつかピックアップ。

### [No.9002 FizzBuzz](https://yukicoder.me/problems/no/9002)
ただのFizzBuzz  

```vim
function! s:main(input) abort
  let result = []

  for i in range(1, a:input[0])
    if i % 15 == 0
      call add(result, 'FizzBuzz')
    elseif i % 5 == 0
      call add(result, 'Buzz')
    elseif i % 3 == 0
      call add(result, 'Fizz')
    else
      call add(result, i)
    endif
  endfor

  return join(result, "\n")
endfunction

let s:input = getline(1, '$')
enew
put =s:main(s:input)
2,$print
```

[実際の提出](https://yukicoder.me/submissions/137342)  

先ほど説明したとおり、入力をバッファからgetlineで取得して、出力は新たにバッファを作成して一旦そこに吐き出してから出力しています。  
実行する場合は、  

```sh
echo '100' | vim -u NONE -i NONE -X -N -n -e -s -S fizzbuzz.vim /dev/stdin -c qa!
```

こんな感じで入力をパイプで渡してやればよいです。

### [No.163 cAPSlOCK](https://yukicoder.me/problems/no/163)
大文字を小文字に、小文字を大文字にする問題  

```vim
norm V~
p
```

[実際の提出](https://yukicoder.me/submissions/138246)  

この問題ではあえて出力用のバッファを作らずに入力用のバッファを編集したものを出力しています。やっていることはnormalコマンドで行選択して大文字小文字を反転させているだけです。  
yukicoderでは他の人の解答も見ることができ、その一覧をコード長順でソートすることができます。この機能を使ってコードゴルフっぽく遊ぶことができ、この解答は[最短コード長を狙ってとったもの](https://yukicoder.me/problems/no/163/submissions?lang_id=&status=AC&sort_length=enabled)(執筆時点)です。Vim scriptでBashやPerlやRuby等の言語に勝てると非常に気持ち良いので、最短コード長を目指してやってみるのもいいと思います。

### [No.3 ビットすごろく](https://yukicoder.me/problems/no/3)
スタートからゴールのマスまで1から順に数字が振られていて、現在いるマスの数字を2進数で表現した時の1のビットの数だけ前後に移動でき、ゴールまで最短何回の移動でたどり着けるか、という問題  

```vim
function! s:output()
  let this = {'lines': []}

  function! this.append(line) abort
    call add(self.lines, a:line)
  endfunction

  function! this.build() abort
    return join(self.lines, "\n")
  endfunction

  return this
endfunction

function! s:main(input) abort
  let o = s:output()

  let n = str2nr(a:input[0])
  let cells = {'1': 1}
  let q = [1]
  let r = n == 1 ? 1 : -1
  while len(q) > 0
    let c = remove(q, 0)
    let b = printf('%b', c)
    let move = len(b) - len(substitute(b, '1', '', 'g'))

    let left = c - move
    let right = c + move

    if left > 0 && !has_key(cells, left)
      let cells[left] = cells[c] + 1
      call add(q, left)
    endif

    if right < n && !has_key(cells, right)
      let cells[right] = cells[c] + 1
      call add(q, right)
    endif

    if right == n
      let r = cells[c] + 1
      break
    endif
  endwhile

  call o.append(r)
  return o.build()
endfunction

let s:input = getline(1, '$')
enew
put =s:main(s:input)
2,$print
```

[実際の提出](https://yukicoder.me/submissions/138028)  

競技プログラミングをやったことない人だとちょっと難しく感じるかもしれませんが、この問題は幅優先探索というアルゴリズムで解くことができます。制約がシビアな問題だと実行速度的にVim scriptでは解けない(制限時間内に処理が終わらない)問題もあると思いますが、この程度のちょっとした幅優先探索の問題ならばVim scriptでも問題なく解けます。


## 問題点
yukicoderでは[リアクティブ形式](https://yukicoder.me/wiki/reactive)という、ジャッジのプログラムと対話的にやり取りをする問題があるのですが、上記のVimの起動コマンドではそれに対応できていません。逐次的に入力を受け取って出力をflushさせる必要があり、その手段がなさそうだったので対応は諦めています... (もし実現できるやり方を知っていたら教えてもらえるとありがたいです)

## まとめ
実際にyukicoderの問題を解いた結果、こんな関数ないかなー？とhelpを眺めたりググって調べたりする機会が増えたので、少しづつVim script力が上がってきた気がします。競技プログラミングをやる言語としてVim scriptは決して向いているとは言えませんが、お遊び程度でやってみる分には面白いと思うので、気になった人は是非やってみてください。


