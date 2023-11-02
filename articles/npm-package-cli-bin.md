---
title: "CLI ツール用の NPM パッケージの作成"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "npm", "cli"]
published: true
---

# 概要

自分への備忘録。
CLI から実行できる NPM パッケージを作成する。

```sh
$ npx my-package

Hello, world!
```

以前に以下の記事で作成した NPM パッケージに追記する形で作成する。

https://zenn.dev/taroshun32/articles/npm-package-original

サンプルコードは以下。

https://github.com/taroshun32/npm-package-sample

# 手順

### ソースコードの実装

```sh
$ mkdir -p src/bin
% touch src/bin/index.ts
```

`Hello, world!` と出力するだけの Node.js スクリプトを用意する。

```typescript:src/bin/index.ts
#!/usr/bin/env node

console.log('Hello, world!')
```

### package.json に CLI 用の設定を追加

bin エントリを指定する。

```json:package.json
"bin": {
  "my-package": "dist/cjs/bin/index.js"
}
```

### tsconfig.esm.json に除外設定を追加

ESM では必要ないため、`exclude` に `src/bin` を追加。(`node_modules`, `dist` も追加する必要がある)

```json:tsconfig.esm.json
"exclude": ["src/bin", "node_modules", "dist"]
```

# ビルドを実行

```sh
$ npm run build:cjs

> my-package@1.0.0 build:cjs
> tsc --project tsconfig.cjs.json
```

dist/cjs/bin ディレクトリ配下に `CLI` 用のファイルが出力されていることを確認する。

```sh
$ tree -a -I 'node_modules'
.
├── dist
│   └── cjs
│       ├── bin
│       │   └── index.js
│       └── index.js
├── ..
```

# 動作確認

package 参照用のリンクを作成する。

```sh
$ npm link my-package

added 1 package, and audited 23 packages in 239ms
found 0 vulnerabilities

$ npm ls --global

/Users/****/.nvm/versions/node/v18.16.0/lib
├── my-package@0.0.0
├── ..
```

実行。

```sh
$ npx my-package

Hello, world!
```

最後にリンクは削除しておく。

```sh
$ npm unlink my-package

removed 1 package, and audited 21 packages in 242ms
found 0 vulnerabilities
```
