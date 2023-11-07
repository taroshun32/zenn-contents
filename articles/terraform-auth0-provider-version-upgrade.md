---
title: "terraform provider auth0 の v1.0.0 がリリース！"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "auth0", "okta"]
published: true
---

# 概要

自分への備忘録。  
先日 terraform-provider-auth0 の v1.0.0 がリリースされ、アップグレード対応を行なったため、対応内容を簡単にまとめる。

https://github.com/auth0/terraform-provider-auth0

今回は v0.45.0 → v1.0.0 へのバージョンアップを実施。
公式のマイグレーションガイドは以下。

https://github.com/auth0/terraform-provider-auth0/blob/main/MIGRATION_GUIDE.md#upgrading-from-v0x--v10

以下 terraform の公式ドキュメントと照らし合わせながらマイグレーションを実施した。

https://registry.terraform.io/providers/auth0/auth0/latest/docs

# 対応内容

## 1. State の管理下から削除

まず breaking changes の中でもあらかじめ terraform の管理外にしておく必要のあるリソースがいくつかあったため、state 管理下からの削除を実行。(リソース名が変わるものはそのままの移行が不可能)  
自分の場合は以下が対象だった。

:::message
ここは人によって異なるため、詳細はマイグレーションガイドを確認すること。
:::

### auth0_email

:::details 変更点
Before (v0.x)
```terraform
resource "auth0_email" "my_email_provider" {
  # ...
}
```

After (v1.0)
```terraform
resource "auth0_email_provider" "my_email_provider" {
  # ...
}
```
:::

### auth0_trigger_binding

:::details 変更点
Before (v0.x)
```terraform
resource "auth0_trigger_binding" "login_flow" {
  #...
}
```

After (v1.0)
```terraform
resource "auth0_trigger_actions" "login_flow" {
  #...
}
```
:::

以下のコマンドで削除を実施。

```sh
$ terraform state rm auth0_email.my_email_provider

Removed auth0_email.my_email_provider
Successfully removed 1 resource instance(s).

$ terraform state rm auth0_trigger_binding.login_flow

Removed auth0_trigger_binding.login_flow
Successfully removed 1 resource instance(s).
```

## 2. バージョンの Upgrade

この状態でバージョンの Upgrade を実施。

```diff terraform
required_providers {
  auth0 = {
    source  = "auth0/auth0"
+   version = "~> 1.0.0"
-   version = "~> 0.45.0"
  }
}
```

以下コマンドを実行。

```sh
terraform init -upgrade
```

## 3. コードの修正

ここからは差分がなくなるまでひたすら愚直にコードの修正を実施。
自分の場合は以下が対象だった。

### auth0_client.client_secret

:::details 変更点
Before (v0.x)
```terraform
resource "auth0_client" "my_client" {
  #...
}

output "my_client_secret" {
  value = auth0_client.my_client.client_secret
}
```

After (v1.0)
```terraform
data "auth0_client" "my_client" {
  client_id = auth0_client.my_client.id
}

output "my_client_secret" {
  value = data.auth0_client.my_client.client_secret
}
```
:::

### auth0_client_grant.scope

:::details 変更点
Before (v0.x)
```terraform
resource "auth0_client_grant" "my_client_grant" {
  #...
  scope = ["create:foo", "create:bar"]
}
```

After (v1.0)
```terraform
resource "auth0_client_grant" "my_client_grant" {
  #...
  scopes = ["create:foo", "create:bar"]
}
```
:::

修正が終われば apply を実施。
(先ほど `state rm` したリソースの差分も出るため、`target-apply` 推奨)

## 4. 削除したリソースの再インポート

先ほど削除したリソースを修正し、インポートを実施。

```sh
$ terraform import auth0_email_provider.my_email_provider ****
data.terraform_remote_state.my_state: Reading...
auth0_email_provider.my_email_provider: Importing from ID "****"...

$ terraform import auth0_trigger_actions.login_flow ****
data.terraform_remote_state.my_state: Reading...
auth0_trigger_actions.login_flow: Importing from ID "****"...
```

これで最終的に差分がなくなれば対応は完了。

# まとめ

個人的に `client_secret` が `data_source` 経由で参照されるようになったのは `state` に残らずよりセキュアになるためいい修正だと感じた。
`auth0_email_provider.credencial` など `apply` が必要となるリソースもあるため、各自慎重に修正する必要はありそう。
