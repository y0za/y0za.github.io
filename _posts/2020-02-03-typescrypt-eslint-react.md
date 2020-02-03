---
layout: post
title: "Reactプロジェクトにtypescript-eslintを導入"
description: ""
date: 2020-02-03
tags: ["eslint", "typescript", "react"]
comments: true
share: true
---

[Getting Started - Linting your TypeScript Codebase](https://github.com/typescript-eslint/typescript-eslint/blob/master/docs/getting-started/linting/README.md)  
ここに書いてあることを参考にしつつ自分のプロジェクトに必要なものを足していく。

### 必要なパッケージ

- `eslint` ESLint 本体
- `eslint-plugin-react` React 用のプラグイン
- `@typescript-eslint/eslint-plugin` TypeScript 用のプラグイン
- `@typescript-eslint/parser` TypeScript 用のパーサ

### ついでに導入したパッケージ

- `eslint-config-prettier` Prettier 用の設定
- `eslint-plugin-prettier` Prettier 用のプラグイン
- `eslint-plugin-react-hooks` React Hooks 用のプラグイン
- `eslint-plugin-jsx-a11y` react-a11y のようなルールを適用するためのプラグイン

### .eslintrc.js

```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint", "react", "react-hooks", "jsx-a11y"],
  extends: [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:@typescript-eslint/eslint-recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:prettier/recommended",
    "prettier/@typescript-eslint"
  ],
  env: { browser: true, es6: true },
  parserOptions: {
    ecmaFeatures: {
      jsx: true
    }
  },
  rules: {
    "react/prop-types": "off",
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
};
```

`react/prop-types` は TypeScript 側でチェックしているので不要。  
`@typescript-eslint/explicit-function-return-type` など制約がキツイ場合は適宜ルールをオフにするといいと思う。

### npm scripts の追加

```json
{
  "scripts": {
    "lint": "eslint src/**/*.{ts,tsx}"
  }
}
```

### その他

[eslint-config-airbnb-typescript](https://www.npmjs.com/package/eslint-config-airbnb-typescript) や [eslint-config-standard-with-typescript](https://www.npmjs.com/package/eslint-config-standard-with-typescript) もあるので、好みに応じてそれらをベースにするとよさそう。
