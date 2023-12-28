---
title: "PR-Agent を使用してプルリクを AI で自動レビューする"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "pragent", "review", "openai", "azureopenai"]
published: true
---

# 概要

PR-Agent は、GitHub のプルリクエスト（PR）に対して自動レビューを行う AI ツールです。

https://github.com/Codium-ai/pr-agent

GitHub Actions を提供しており、プルリクの作成時や、コメントに以下のようなコマンドを入力することで起動できます。

- `/descrive`: PR のタイトルと説明を更新
- `/review`: コードレビュー
- `/improve`: コードの改善提案
... etc

今回はプルリク作成時に `/descrive` & `/review` 、PR コメントで `/improve` を実行するように構築してみます。

:::message
まだ安定版がリリースされていないため、この記事では最新の `v0.11` を使用します。
:::

# 構築

では実際に [公式ドキュメント](https://github.com/Codium-ai/pr-agent/blob/main/Usage.md) を元に GitHub Actions を構築していきます。
まず workflow のベースを作成します。

```yml: .github/workflows/pr-agent.yaml
name: pr-agent

on:
  pull_request:
    types: [opened]
  issue_comment:
    types: [created, edited]

permissions:
  pull-requests: write
  issues:        write

jobs:
  pr_agent:
    runs-on: ubuntu-latest
    name: Run PR Agent
    if: ${{ github.event.sender.type != 'Bot' }}
    steps:
      - id: pr-agent
        uses: Codium-ai/pr-agent@v0.11
        env:
          OPENAI_KEY:   ${{ secrets.OPENAI_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          github_action.auto_describe: true
          github_action.auto_review:   true
          github_action.auto_improve:  false
```

今回は `/descrive` & `/review` を PR 作成時に自動で実行するように設定しています。

次に env セクションに環境変数を追加して各種設定を行なっていきます。
OpenAI の GPT4 の場合はこのままで良いのですが、AzureOpenAI など他のモデルを使用する場合は [ドキュメント](https://github.com/Codium-ai/pr-agent/blob/main/Usage.md#azure) にあるように追加の設定が必要です。
今回は AzureOpenAI を使用するため、以下の環境変数を追加します。

```diff yml
env:
    OPENAI_KEY: ${{ secrets.OPENAI_KEY }}
    ...
+   OPENAI.API_TYPE:      "azure"
+   OPENAI.API_BASE:      ${{ secrets.OPENAI_API_BASE }}
+   OPENAI.API_VERSION:   ""
+   OPENAI.DEPLOYMENT_ID: ""
+   CONFIG.MODEL:         ""
```

後は日本語対応も行っておきましょう。
`extra_instructions` を追加し、日本語対応する旨を記述します。

```diff yml
env:
    OPENAI_KEY: ${{ secrets.OPENAI_KEY }}
    ...
+   PR_DESCRIPTION.EXTRA_INSTRUCTIONS:      'Please use Japanese.'
+   PR_REVIEWER.EXTRA_INSTRUCTIONS:         'Please use Japanese.'
+   PR_CODE_SUGGESTIONS.EXTRA_INSTRUCTIONS: 'Please use Japanese.'
```

他にも設定項目は多岐に渡るため、[公式のコード](https://github.com/Codium-ai/pr-agent/blob/main/pr_agent/settings/configuration.toml) を確認しながら色々と試してみると良いと思います。

# 実行

ここまで出来たら実際に自動レビューを実行してみましょう。
まずは特にコード等追加せずこのまま PR を作成してみます。

作成後 Action が稼働し、しばらくすると PR の説明とレビューが追加されるはずです。

`/describe`
![](/images/pr-agent-pull-request-ai-review/review-1.png)

`/review`
![](/images/pr-agent-pull-request-ai-review/review-2.png)

良い感じですね。
日本語対応も問題なく出来てそうです。

Issue Comment のイベント実行はデフォルトブランチにワークフローが実装されている必要があるため、一度 main ブランチにマージし、次はあえて冗長な配列処理のコードを記述した typescript のファイルを追加してみます。

::::details 追加コード
```ts: test.ts
// 1. 配列の要素を追加・削除する -----------------------------------------------------------------------------------------

// 配列の初期化
let fruits: string[] = ["Apple", "Banana", "Orange"];

// 要素の追加
let newFruit = "Grape";
fruits = [...fruits, newFruit];

// 要素の削除
let removedFruitIndex = 1;
fruits = [...fruits.slice(0, removedFruitIndex), ...fruits.slice(removedFruitIndex + 1)];

// 2. 配列をフィルタリングする -------------------------------------------------------------------------------------------

// 数字の配列
let numbers: number[] = [1, 2, 3, 4, 5, 6];

// 奇数のみをフィルタリング
let oddNumbers: number[] = [];
for (let i = 0; i < numbers.length; i++) {
    if (numbers[i] % 2 !== 0) {
        oddNumbers.push(numbers[i]);
    }
}

// 3. 配列の各要素に関数を適用する ----------------------------------------------------------------------------------------

// 文字列の配列
let names: string[] = ["Alice", "Bob", "Charlie"];

// 各要素を大文字に変換
let uppercaseNames: string[] = [];
for (let i = 0; i < names.length; i++) {
    uppercaseNames.push(names[i].toUpperCase());
}

// 4. 配列の要素を合計する ----------------------------------------------------------------------------------------------

let values: number[] = [10, 20, 30, 40];

// 合計値を計算
let sum = 0;
for (let i = 0; i < values.length; i++) {
    sum += values[i];
}

// 5. 配列の検索 -------------------------------------------------------------------------------------------------------

let colors: string[] = ["red", "green", "blue"];

// 要素の検索
let indexOfGreen = -1;
for (let i = 0; i < colors.length; i++) {
    if (colors[i] === "green") {
        indexOfGreen = i;
        break;
    }
}
```
::::

PR 作成後、`/improve` とコメントすると絵文字が付き、Action が終了すると以下のようなインラインコメントがいくつか生成されています。

![](/images/pr-agent-pull-request-ai-review/comment.png)

![](/images/pr-agent-pull-request-ai-review/review-3.png)

# まとめ

今回は簡単なコードで試してみただけですが、使い勝手はすごく良い感じです。
安定版のリリースが待ち遠しいですね。

実際のプロダクトなどの場合は使いすぎると結構なトークン数に達してしまい料金も跳ね上がりそうなので、導入には注意が必要です。

## 参考

- [PR-Agent を使って Pull Request をAIレビューしてみた（日本語対応もしてみた）](https://tech.layerx.co.jp/entry/2023/09/01/102612)
