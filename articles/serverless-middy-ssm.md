---
title: "[serverless] @middy/ssm を用いて AWS パラメータストアから値を取得する "
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","Serverless","パラメータストア","middy","TypeScript"]
published: true
---

## 背景

今まで Serverless からパラメータストアの値を取得する際は AWS Parameters and Secrets Lambda Extension を使用していたのですが、middy というミドルウェアエンジンを使用するとさらにコードを簡略化することができたので紹介したいと思います。

## 前提

Serverless framework と環境構築手順に関しては以下公式ドキュメントを参照してください。

https://www.serverless.com/framework/docs

## middy とは

https://middy.js.org/

- Lambda にミドルウェア処理を追加することに特化した、シンプルで軽量なライブラリ。
- エラーハンドリングやバリデーションなど、ビジネスロジック以外の処理を分離し、複数ハンドラ間で利用できるよう設計されている。

実際に公式から提供されているミドルウェアの一覧は以下から確認できます。

https://middy.js.org/docs/middlewares/intro/

今回は詳しく説明しませんが、以下公式のサンプルコードのように、ミドルウェアを自作して特定のフェーズで実行する処理を記述することもできます。
独自のエラーハンドラを実装したい場合は `customMiddlewareOnError` 内に処理を記述していくイメージです。

```diff ts:custom-middleware.ts
const defaults = {}

const customMiddleware = (opts) => {
  const options = { ...defaults, ...opts }

  const customMiddlewareBefore = async (request) => {
    const { event, context } = request
    // ...
  }

  const customMiddlewareAfter = async (request) => {
    const { response } = request
    // ...
    request.response = response
  }

  const customMiddlewareOnError = async (request) => {
    if (request.response === undefined) return
    return customMiddlewareAfter(request)
  }

  return {
    before:  customMiddlewareBefore,
    after:   customMiddlewareAfter,
    onError: customMiddlewareOnError
  }
}

export default customMiddleware
```

この記事では公式から提供されている `@middy/ssm` を使用してパラメータストアから値を取得してみたいと思います。

https://middy.js.org/docs/middlewares/ssm/

## 実装

では実際にサンプルプロジェクトを作成していきます。
今回は `aws-nodejs-typescript (v3)` のテンプレートを使用します。

```
> serverless create --template aws-nodejs-typescript --path middy-ssm-sample

✔ Project successfully created in "middy-ssm-sample" from "aws-nodejs-typescript" template (2s)

> tree .
.
├── README.md
├── package.json
├── serverless.ts
├── src
│   ├── functions
│   │   ├── hello
│   │   │   ├── handler.ts
│   │   │   ├── index.ts
│   │   │   ├── mock.json
│   │   │   └── schema.ts
│   │   └── index.ts
│   └── libs
│       ├── api-gateway.ts
│       ├── handler-resolver.ts
│       └── lambda.ts
├── tsconfig.json
└── tsconfig.paths.json

5 directories, 13 files
```

こちらのテンプレートでは始めから middy はインストールされているため、今回使用する `@middy/ssm` `@aws-sdk/client-ssm` をインストールします。

```
> npm install --save @middy/ssm
> npm install --save-dev @aws-sdk/client-ssm
```

`src > libs > lambda.ts` に初期化処理は記述されているため、ここに `@middy/ssm` を使用してパラメータストアから値を取得するための処理を追記していきます。取得した値は `request.context` から使用できるように設定します。

- `fetchData`: パラメータの Name or Path
- `setToContext`: 取得した値を `request.context` に保存するかどうか

```diff ts:lambda.ts
import middy from "@middy/core"
import middyJsonBodyParser from "@middy/http-json-body-parser"

import ssm from "@middy/ssm" // 追記

export const middyfy = (handler) => {
  return middy(handler)
    .use(middyJsonBodyParser())
    .use(ssm({                          // 追記
      fetchData: {                      // 追記
        secretValue1: 'secret_value_1', // 追記
        secretValue2: 'secret_value_2'  // 追記
      },                                // 追記
      setToContext: true                // 追記
    }))                                 // 追記
}
```

次に、パラメータストアから値を取得するための iam_role の設定を行います。
必要となる permission は以下です。

- `kms:Decrypt`
- `ssm:GetParameters` or `ssm:GetParametersByPath`

今回は例として terraform で作成したものを `serverless.ts` に設定していきます。

```diff ts:serverless.ts
  provider: {
    name: 'aws',
    runtime: 'nodejs14.x',
    apiGateway: {
      minimumCompressionSize: 1024,
      shouldStartNameWithService: true,
    },
    environment: {
      AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1',
      NODE_OPTIONS: '--enable-source-maps --stack-trace-limit=1000',
    },
    iam: { role: 'arn:aws:iam::************:role/serverless_role'}, // 追記
  },
```

```diff tf:iam.tf
resource "aws_iam_role" "serverless_role" {
  name               = "serverless_role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_policy" "serverless_policy" {
  name   = "serverless_policy"
  policy = data.aws_iam_policy_document.serverless_policy.json
}

data "aws_iam_policy_document" "serverless_policy" {
  statement {
    sid = "1"
    actions = [
      "ssm:GetParameters",
      "ssm:GetParametersByPath",
    ]
    resources = [
      "arn:aws:ssm:ap-northeast-1:************:parameter/＊"
    ]
  }

  statement {
    sid = "2"
    actions = [
      "kms:Decrypt"
    ]
    resources = [
      "*",
    ]
  }
}

resource "aws_iam_role_policy_attachment" "serverless_policy" {
  role       = aws_iam_role.serverless_role.name
  policy_arn = aws_iam_policy.serverless_policy.arn
}
```

あとは実際にハンドラから context 経由でパラメータを呼び出せれば実装は完了です。
非常に簡単ですね。

```diff ts:handler.ts
async function hello(event, context) {
  console.log(context.SECRET_VALUE_1)
  console.log(context.SECRET_VALUE_2)
}

export const main = middyfy(hello)
```

## まとめ

今回は `@middy/ssm` を使用してパラメータストアの値を取得してみました。
今後は他の middy のミドルウェアであったり、自作のエラーハンドラなどを作成してみたいなと思います。
