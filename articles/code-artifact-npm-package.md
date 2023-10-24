---
title: "AWS CodeArtifact で 独自 NPM パッケージを管理する"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["codeartifact", "aws", "typescript", "npm", "terraform"]
published: false
---

# 概要

自分への備忘録。
CodeArtifact でプライベートリポジトリとして独自パッケージを管理するための手順。
AWS 部分は terraform で構築する。

以前に別記事で作成した以下の NPM パッケージを使用する。

https://github.com/taroshun32/npm-package-sample
※ ここにリンクを設定

# CodeArtifact を作成

以下の名前で作成する。

- ドメイン名 `my-domain`
- リポジトリ名 `my-repository`
- アップストリーム 無し

:::message
アップストリームに関して

この機能により、CodeArtifact は maven や npm などの対応するパッケージマネージャーの Proxy として利用できるが、別途料金がかかるため今回は無効とし、後述する `.npmrc` の設定によって取得元を判別するように制御する。
:::

```tf:code-artifact.tf
resource "aws_kms_key" "codeartifact_key" {
  description             = "CMK For CodeArtifact"
  deletion_window_in_days = 7
}

resource "aws_kms_alias" "codeartifact_key" {
  name          = "alias/my-codeartifact-key"
  target_key_id = aws_kms_key.codeartifact_key.key_id
}

resource "aws_codeartifact_domain" "my_domain" {
  domain         = "my-domain"
  encryption_key = aws_kms_alias.codeartifact_key.arn
}

resource "aws_codeartifact_repository" "my_repository" {
  repository = "my-repository"
  domain     = aws_codeartifact_domain.my_domain.domain
}
```

今回は細かい権限の設定は省略する。
ドメインやリポジトリ単位でポリシーの設定が可能であり、クロスアカウントでのインストールなども可能である。

# 名前空間の設定

デフォルトでは、npm は標準 registry (npmjs.com) に対して接続しようとする。
しかし今回の場合はアップストリームを無効にしており、 CodeArtifact で管理してるパッケージだけは CodeArtifact に接続する必要があるため、名前空間を設定する。

package.json を以下のように修正する。

```json:package.json
"name": "@my-namespace/my-package"
```

名前空間を設定することにより、以下のように `.npmrc` の `registry` 設定を行うことで `@my-namespace` の名前空間が付与されたパッケージのみが CodeArtifact に接続されるようになる。

```:.npmrc
@my-namespace:registry=https://my-domain-************.d.codeartifact.ap-northeast-1.amazonaws.com/npm/my-repository/
```

# 認証トークンの設定

CodeArtifact では、IAM の認証情報だけではリポジトリアクセスはできず、別途 CodeArtifact にログインし、認証トークンの設定を行う必要がある。

まず CodeArtifact 認証トークンをエクスポートする。

```sh
export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain my-domain --domain-owner ************ --region ap-northeast-1 --query authorizationToken --output text`
```

次に `.npmrc` を設定する。

```:.npmrc
@my-namespace:registry=https://my-domain-************.d.codeartifact.ap-northeast-1.amazonaws.com/npm/my-repository/
//my-domain-************.d.codeartifact.ap-northeast-1.amazonaws.com/npm/my-repository/:_authToken=${CODEARTIFACT_AUTH_TOKEN}
```

# 動作検証

ここまで来れば CodeArtifact に対して `publish`, `install` ができるようになる。

```sh
npm publish
```

```sh
npm install @my-repository/my-package@1.0.0
```

CLI が無事実行できれば OK。

```sh
$ npx my-package

Hello, world!
```

# おまけ

認証トークンの設定を都度手動で行うのは手間なため、設定用のシェルスクリプトを用意すると良さそう。

```sh:code-artifact-setup.sh
#!/bin/bash

# CodeArtifactの認証トークンを取得
CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain my-domain --domain-owner ************ --region ap-northeast-1 --query authorizationToken --output text`

# registryを設定
npm config set @my-namespace:registry=https://my-domain-************.d.codeartifact.ap-northeast-1.amazonaws.com/npm/my-repository/

# 認証トークンを設定
npm config set //my-domain-************.d.codeartifact.ap-northeast-1.amazonaws.com/npm/my-repository/:_authToken=${CODEARTIFACT_AUTH_TOKEN}

```

これで作業前はシェルスクリプトの実行を行うだけで設定ができるようになる。

```sh
sh code-artifact-setup.sh
```
