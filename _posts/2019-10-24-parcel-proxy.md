---
layout: post
title: "parcelの開発環境でプロキシを設定する"
description: ""
date: 2019-10-24
tags: ["parcel", "JavaScript"]
comments: true
share: true
---

[parcel](https://parceljs.org/)で開発を行う上でAPIサーバなどへの一部のリクエストをプロキシさせたい場合の方法。


[webpack](https://webpack.js.org/)の場合は[webpack-dev-server](https://github.com/webpack/webpack-dev-server)に[proxy](https://webpack.js.org/configuration/dev-server/#devserverproxy)があるが、parcelは単純なバンドラーなのでそういった機能は用意されていない。  
なので、[express](https://expressjs.com/)などでサーバを立ててparcelへのリクエストとプロキシさせたいリクエストを振り分けるようにする。

パッケージの追加
```console
$ npm install -D express http-proxy-middleware
# もしくは
$ yarn add -D express http-proxy-middleware
```


以下のようなコードをdev.jsとして用意する。
この場合 `/api` へのリクエストを `http://localhost:8080/api` にプロキシさせている。
```js
const proxy = require('http-proxy-middleware');
const Bundler = require('parcel-bundler');
const express = require('express');

const bundler = new Bundler('./index.html');

const app = express();

app.use(
  '/api',
  proxy({
    target: 'http://localhost:8080',
  }),
);

app.use(bundler.middleware());

app.listen(Number(process.env.PORT || 1234));
```


package.jsonのscriptsは以下のようにすればよい。
```json
{
  "scripts": {
    "dev": "node dev.js",
    "build": "parcel build ./index.html"
  }
}
```


### 参考
https://github.com/parcel-bundler/parcel/issues/55#issuecomment-357327211
