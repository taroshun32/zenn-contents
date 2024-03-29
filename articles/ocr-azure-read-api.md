---
title: "手書きの日本語メモ画像から文字を抽出してみる (Azure AI Vision)"
emoji: "👁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "azure", "ocr"]
published: true
---

# 概要

自分への備忘録。
Azure AI Vision (v3.2) の Read API を使用して画像から日本語文字列の抽出 (OCR 処理) を行う。
今回は Azure 周りのリソースの作成は省略し、Node.js SDK を使用した実装方法を解説する。

#### 公式ドキュメント
https://learn.microsoft.com/ja-jp/azure/ai-services/computer-vision/overview-ocr

#### サンプルコード
簡易的な nodejs の実行環境を用意済み。
https://github.com/taroshun32/ocr-with-azure-read-api

# 前提

Azure の Computer Vision を作成し、以下を取得しておく。
- API キー
- エンドポイント

https://portal.azure.com/#create/Microsoft.CognitiveServicesComputerVision

# 手順

### ライブラリのインストール

まず必要となる以下2つのライブラリをインストールする。

#### [@azure/cognitiveservices-computervision](https://www.npmjs.com/package/@azure/cognitiveservices-computervision/v/8.2.0)
Azure の Computer Vision API を使用するための Node.js クライアントライブラリ。
このライブラリを使用して OCR 処理を行う。

#### [@azure/ms-rest-js](https://www.npmjs.com/package/@azure/ms-rest-js)
Azure の RESTful サービスと通信するためのライブラリ。
HTTP リクエスト周りの処理や、API キー認証など Azure サービスとの認証を行う。

```sh
$ npm install @azure/cognitiveservices-computervision
$ npm install @azure/ms-rest-js
```

### 実装

最終的なコードは以下。

::::details main.ts
https://github.com/taroshun32/ocr-with-azure-read-api/blob/main/src/index.ts
::::

まず OCR 処理を行う画像ファイルを Buffer として取得する。
`fs` にはファイルを直接 Buffer として読み込む `fs.readFileSync` メソッドが用意されているが、今回はサイズの大きいファイルにも対応できるよう `fs.createReadStream` メソッドを使用して ReadableStream を作成し、自前で用意した `streamToBuffer` 関数で Buffer に変換している。

```ts
async function main() {
  ...
  const imageStream = fs.createReadStream(FILE_PATH)
  const imageBuffer = await streamToBuffer(imageStream)
  ...
}

async function streamToBuffer(readableStream: Readable): Promise<Buffer> {
  const chunks: any[] = []
  for await (const chunk of readableStream) {
    chunks.push(chunk)
  }
  return Buffer.concat(chunks)
}
```

次に `ComputerVisionClient` の `readInStream` メソッドを使用して OCR 処理のリクエストを行う。
この時点で OCR の結果が返ってくるわけではなく、レスポンスとして `OperationLocation` が返されるため、この URL から後続処理 (OCR 結果の取得) で必要となる `operationId` を取得する。

```ts
const readInStreamResult = await azure.readInStream(imageBuffer, { language: 'ja' })
const operationId        = readInStreamResult.operationLocation.split('/').pop()!
```

Azure の OCR 機能は画像からテキストを読み取る処理を非同期に行うため、処理が完了するまで一定の間隔で実行ステータスの確認 (ポーリング) を行う必要がある。
ここでは `pollUntilSucceeded` 関数を使用して5秒間隔で実行ステータスを確認し、完了次第結果を取得するよう実装している。

```ts
async function main() {
...
  const ocrResult = await pollUntilSucceeded(azure, operationId)
...
}

async function pollUntilSucceeded(
  azure:       ComputerVisionClient,
  operationId: string
): Promise<GetReadResultResponse> {
  const result = await azure.getReadResult(operationId)
  console.debug(`ocr: ${result.status}`)
  
  if (result.status !== 'succeeded') {
    await new Promise(resolve => setTimeout(resolve, 5000))
    return pollUntilSucceeded(azure, operationId)
  }
  return result
}
```

最後に必要に応じて OCR 処理の結果を加工すれば実装は完了。

```ts
const ocrText = ocrResult.analyzeResult!.readResults.reduce((text, page) => {
  const pageText = page.lines.reduce((lineText, line) => lineText + line.text + '\n', '')
  return text + pageText
}, '')

console.debug(inspect({ ocrText }))
```

### 動作確認

メモ帳に雑に書いた以下のメモ画像を読み取らせてみると、正確に読み取れていることが確認できる。

![](/images/ocr-azure-read-api/ocr-sample.png)

```sh
$ npm run start

ocr: running
ocr: succeeded
{ ocrText: 'こんにちは。\n私の名前はTaroshun32です。\nOCRについて勉強中です。\n' }
```
