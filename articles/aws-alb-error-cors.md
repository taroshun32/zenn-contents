---
title: "[ALB] CORS に対応したメンテナンスモード(503)を構築する"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","alb","cors","メンテナンス","terraform"]
published: true
---

## 概要

自分への備忘録。  
ALB のリスナールールで 503 を返却することでメンテナンスモードを実現する際、CORS ヘッダーの対応が必要な場合の対応方法。  
Terraform と Serverless を用いてサクッと実現する。

## CORS 対応が必要ない場合

直接 503 を返却すれば問題ない。  
レスポンスでメッセージを返却したい場合は json で対応可能。

```tf:alb.tf
resource "aws_lb_listener_rule" "maintenance" {
  ...

  action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      status_code  = "503"
    }
  }
}
```

## CORS 対応が必要な場合

ALB 単体での固定レスポンスではヘッダーが設定できないため、ターゲットに Lambda 関数を指定する。

Lambda は Serverless でサクッと構築する。

```sh
$ npm install -g serverless

$ serverless create --template-url https://github.com/serverless/examples/tree/v3/aws-node-typescript --path maintenance-resolver
```

handler.ts を編集する。  

```ts: handler.ts
export async function maintenance(event) {
  // pre-flight リクエスト時には 204 を返す
  if(event.httpMethod === 'OPTIONS') {
    return {
      statusCode: 204,
      headers: {
        'Content-Type': 'application/json; charset=utf-8',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': '*',
        'Access-Control-Allow-Credentials': 'true'
      },
      body: ''
    }
  }

  const message = process.env.MAINTENANCE_MESSAGE || 'ただいまメンテナンス中です。'
  return {
    statusCode: 503,
    headers: {
      'Content-Type': 'application/json; charset=utf-8',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Headers': '*',
      'Access-Control-Allow-Credentials': 'true'
    },
    body: JSON.stringify({"message": message})
  }
}
```

都度デプロイは面倒なため、メッセージは環境変数から設定できるようにしておく。  
serverless.yml をよしなに編集してデプロイ。

```sh
$ sls deploy --verbose
```

次に Terraform で以下を作成する。
- aws_lb_target_group
- aws_lb_listener_rule

まずはターゲットグループを作成。  
lambda のアタッチまで一括で行う。

```tf:alb.tf
resource "aws_lb_target_group" "maintenance" {
  name        = "maintenance"
  target_type = "lambda"

  ...
}

resource "aws_lambda_permission" "maintenance" {
  action        = "lambda:InvokeFunction"
  function_name = data.aws_lambda_function.maintenance.arn
  principal     = "elasticloadbalancing.amazonaws.com"
  source_arn    = aws_lb_target_group.maintenance.arn
}

data "aws_lambda_function" "maintenance" {
  function_name = "maintenance-resolver"
}

resource "aws_lb_target_group_attachment" "maintenance" {
  target_group_arn = aws_lb_target_group.maintenance.arn
  target_id        = data.aws_lambda_function.maintenance.arn
  depends_on       = [aws_lambda_permission.maintenance]
}
```

最後に ALB にリスナールールを追加すれば完了。

```tf:alb.tf
resource "aws_lb_listener_rule" "maintenance" {
  ...

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.maintenance.arn
  }
}
```

コード管理も簡潔にできていいですね。

## 参考

- [APIを提供するALBでメンテナンスモード(503)を実現する(CORSに対応する)](https://qiita.com/unotovive/items/21ef1d42b01f4e35df4c)
