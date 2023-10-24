---
title: "[TypeScript] 独自 NPM パッケージの作成"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "npm", "commonjs", "esmodules"]
published: false
---

# 概要

自分への備忘録。  
社内ツールとして NPM パッケージを自作した時の知見をまとめる。 
DualPackage (CJS/ESM) 対応する。

```javascript:JavaScript
const { hello } = require('my-package');
hello();  // 'Hello, world!'
```

```typescript:TypeScript
import { hello } from 'my-package';
hello();  // 'Hello, world!'
```

サンプルコードは以下。

https://github.com/taroshun32/npm-package-sample

# 環境構築

### Node.js として初期化

```sh
$ mkdir my-package
$ cd my-package

$ npm init --yes
```

package.json が作成される。

```sh
$ tree
.
└── package.json

1 directory, 1 file
```

### TypeScript をインストール

```sh
npm install --save-dev typescript ts-node @types/node
```

# ビルド設定

### CJS 用の tsconfig.cjs.json を作成

CJS モジュールとして出力するための設定を記述する。  
dist/cjs ディレクトリに出力する。

```json:tsconfig.cjs.json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "esModuleInterop": true,
    "strict": true,
    "lib": ["ESNext"],
    "types": ["node"],
    "outDir": "dist/cjs"
  },
  "include": ["src"]
}
```

### ESM 用の tsconfig.esm.json を作成

ESM モジュールとして出力するための設定を記述する。  
dist/esm ディレクトリに出力する。

```json:tsconfig.esm.json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Node",
    "esModuleInterop": true,
    "strict": true,
    "lib": ["ESNext"],
    "types": ["node"],
    "outDir": "dist/esm",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src"]
}
```

### package.json を修正

`CJS`, `ESM` それぞれの Build 用のスクリプトを用意する。

```json:package.json
{
  "scripts": {
    "build:cjs": "tsc --project tsconfig.cjs.json",
    "build:esm": "tsc --project tsconfig.esm.json"
  },
}
```

`CJS`, `ESM` それぞれの読み込み先ファイルの設定を追加する。

```json:package.json
{
  "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "types": "dist/esm/index.d.ts",
  "files": [
    "dist"
  ]
}
```

# ソースコードの実装

```sh
$ mkdir src
$ touch src/index.ts
```

`Hello, world!` と出力するだけの `hello` 関数を用意する。

```typescript:src/index.ts
export const hello = (): void => {
  console.log('Hello, world!')
}
```

# ビルドを実行

まずは `CJS`。

```sh
$ npm run build:cjs

> my-package@1.0.0 build:cjs
> tsc --project tsconfig.cjs.json
```

dist/cjs ディレクトリ配下に `CJS` 用のファイルが出力されていることを確認する。

```sh
$ tree -a -I 'node_modules'
.
├── dist
│   └── cjs
│       └── index.js
├── ..
```

次に `ESM`。

```sh
$ npm run build:esm

> my-package@1.0.0 build:esm
> tsc --project tsconfig.esm.json
```

dist/esm ディレクトリ配下に `ESM` 用のファイルが出力されていることを確認する。

```sh
$ tree -a -I 'node_modules'
.
├── dist
│   ├── cjs
│   │   └── index.js
│   └── esm
│       ├── index.d.ts
│       ├── index.d.ts.map
│       └── index.js
├── ..
```

ここまで問題なければ NPM パッケージの作成は完了。  
今後は AWS CodeArtifact での管理方法や Rollup を用いた `.cjs`, `.mjs` 拡張子でのビルド方法などを別記事でまとめてみたい。
