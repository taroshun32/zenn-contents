---
title: "GitHub API 経由で GitHub Actions の repository_dispatch イベントを実行する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHubActions","RepositoryDispatch"]
published: false
---

## 背景

プロジェクトの CI/CD を構築する際、Slack での確認ボタンを経由しつつ、外部から API 経由で GitHub Actions を実行したいという要件がありました。
フローで言うと下記画像のような感じです。

![](/images/repository-dispatch/flow.png)

そこで今回は Lambda から API 経由で repository_dispatch という webhook イベントをトリガーすることでこれを実現しました。
このフローでは Lambda からトリガーしていますが、API を叩ける環境であれば何でも大丈夫です。

## 前提

GitHub Actions に関しては以下を参照してください。
https://github.co.jp/features/actions

## repository-dispatch イベントを作成

まずは Github Actions の Workflow を作成していきます。
Workflow の yml ファイルに以下の設定を記述します。
```
on:
  repository_dispatch:
    types:
      - deploy
```
types は API リクエストする際に指定する `event_type` 名で、任意の名前を設定できます。
後述しますが、repository_dispatch では API リクエストの際 `client_payload` というパラメータを送信することができ、workflow 側では `github.event.client_payload` コンテキストで使用できます。
```
on:
  repository_dispatch:
    types:
      - deploy

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - env:
          MESSAGE: ${{ github.event.client_payload.message }}
        run: echo $MESSAGE
```

## GitHub との外部連携

GitHubには、外部連携を行う方法として以下の仕組みがあります。
- Personal Access Token
- GitHub Apps

Personal Access Token は個人ユーザーに紐づくため、社員が退職したりアカウントが削除されるとトークンも削除されてしまいます。
GitHub Apps は Organization 単位で使用することができるため、今回はこちらを使用して外部から API を実行します。

## GitHub Apps を登録

実際に使用できる状態までの流れとしては以下となります。
1. GitHub Apps の作成
2. GitHub Apps をインストール (権限設定)

### 1. GitHub Apps の登録

GithHub Apps は User、もしくは Organization 単位で登録できます。
以下の公式ドキュメントを参考に、
`Setting > Developer Settings > GitHub Apps`
から登録してください。
https://docs.github.com/ja/apps/creating-github-apps/registering-a-github-app/registering-a-github-app

### 2. GitHub Apps をインストール (権限設定)

GitHub Apps を作成したら、それをインストールする必要があります。
`Setting > Developer Settings > GitHub Apps > Edit`
から設定ページを開いてインストールし、Workflow を作成したリポジトリにアクセス許可設定をしてください。
必要な Permission は以下となります。
- `metadata:read`
- `contents:read&write`

また、その際 Private key を生成し、App ID と共にメモしておいてください。
後の手順で外部認証する際に必要となります。 
https://docs.github.com/ja/apps/using-github-apps/installing-your-own-github-app


## アクセストークンを取得
これで API リクエストをする準備が整ったので、今回は例として TypeScript を用いて、実際に Workflow をトリガーする処理を書いていきます。
まずは外部から GitHub App をして認証を通すため、以下の手順でアクセストークンを取得する必要があります。


1. Json Web Token (JWT) を生成
2. Installation ID を取得
3. アクセストークン を取得

https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app

### 1. Json Web Token (JWT) を生成

今回は `jsonwebtoken` と `axios` を使います。
まずは JWT の Payload を作成していきます。
GitHub の App ID と Secret key は環境変数に格納しています。
- iat: 発行時刻
- exp: 失効時刻
- iss: GitHub App ID
```
const payload = {
  exp: Math.floor(Date.now() / 1000) + 60,
  iat: Math.floor(Date.now() / 1000) - 10,
  iss: process.env.GITHUB_APP_ID
}
```

次に `jsonwebtoken` を使用して Secret key で署名し、JWT を作成していきます。
```
import { sign } from 'jsonwebtoken'

const secret = Buffer.from(process.env.GITHUB_SECRET_KEY)
const jwt    = sign(payload, secret, { algorithm: 'RS256'})
```

### 2. Installation ID を取得

次に Installation ID を取得していきます。
GitHub App には特定の user や organization にインストールされていることを表す Installation と言う概念があり、 installation ID 毎にアクセストークンが発行されるため、この installation ID を特定する必要があります。
先ほどの JWT を用いて API でリクエストします。
```
import axios from 'axios'

const response = await axios.get(
  'https://api.github.com/app/installations',
  {
    headers: {
      Authorization: 'Bearer ' + jwt,
      Accept:        'application/vnd.github.v3+json'
    }
  }
)

const installationId = response.data[0].id
```

### 3. アクセストークン を取得

次に Installation ID を用いてアクセストークンを取得します。
ここでも先ほどと同様に JWT を用いて API でリクエストします。
```
const response = await axios.post(
  `https://api.github.com/app/installations/${installID}/access_tokens`,
  null,
  {
    headers: {
      Authorization: 'Bearer ' + jwt,
      Accept:        'application/vnd.github.v3+json'
    }
  }
)

const accessToken = response.data.token
```

## イベントを dispatch

ここまで来ればあとはアクセストークンを使用して実際に repository_dispatch イベントをトリガーするだけです。
前の手順で作成した Workflow の `event_type` と 任意の `client_payload` をパラメータとして設定します。
```
await axios.post(
  'https://api.github.com/repos/{owner}/{repository}/dispatches',
  {
    event_type: 'ecs-deploy',
    client_payload: {
      MESSAGE1: 'message 1',
      MESSAGE2: 'message 2'
    }
  },
  {
    headers: {
      Authorization: 'token ' + accessToken,
      Accept:        'application/vnd.github.v3+json'
    }
  }
)
```

これで無事 Actions が実行されていれば構築は完了です。
