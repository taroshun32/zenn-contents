---
title: "MastraのMemory機能を深掘る"
emoji: "💬"
type: "tech"
topics: ["mastra", "ai", "aiエージェント", "nodejs", "dynamodb"]
published: false
---


# 概要

AIチャットボットやエージェントを構築する際、最も重要な要素の一つが「記憶」です。ユーザーとの過去の会話を記憶し、文脈を理解した上で継続利用するためには、適切なメモリシステムが不可欠です。

筆者も最近、業務効率化や社内AIエージェント構築を行う機会が増えていますが、毎回メモリシステムの構築に苦戦しています。そこで今回は、MastraのMemory機能を詳しく深掘りし、実践的な実装パターンまで含めて理解を深める目的でまとめてみます。

https://mastra.ai/ja/docs/memory/overview

:::message
本記事は以下のバージョンに基づいて執筆しています。
- `@mastra/memory`: v0.12.1
- `@mastra/core`: v0.13.1
- `@mastra/dynamodb`: v0.14.0
:::

# Mastra Memoryとは

Mastra Memoryは、AIエージェントの会話管理に特化した包括的なライブラリです。以下の4つの主要機能を提供します。

1.  **会話履歴の管理**  - メッセージの保存・取得・管理
2.  **Working Memory**  - セッション横断的な永続記憶
3.  **セマンティック検索**  - 関連する過去の会話を検索
4.  **プロセッサー**  - メッセージの前処理・最適化

# アーキテクチャ

## 3層構造

```
┌─────────────────────┐
│       Memory　　   　│ ← 上記4つの主要機能を提供する高レベルAPI
├─────────────────────┤
│       Storage　　  　│ ← メッセージの永続化
├─────────────────────┤
│    Vector Store　　　│ ← 過去の会話のセマンティック検索(オプション)
└─────────────────────┘
```

## コンポーネントの責務

### Memory層

会話管理の複雑な処理を抽象化し、シンプルなAPIを提供。

### Storage層

データの永続化を担当。複数のアダプターから選択可能。

-   `@mastra/dynamodb`  - AWS DynamoDB
-   `@mastra/postgres`  - PostgreSQL
-   `@mastra/upstash`  - Upstash Redis
-   `@mastra/libsql`  - LibSQL
- その他多数のアダプターに対応

### Vector Store層（オプション）

直近の履歴から外れた、過去の会話のセマンティック検索のためのベクトルデータベース。

# 各主要機能の詳細

## 1. 会話履歴の管理

会話履歴の管理は、Mastra Memoryの最も基本的な機能です。過去のメッセージを自動的に保存し、新しいリクエスト時に適切な数の履歴（デフォルト10件）をコンテキストに含めます。これにより、AIが文脈を理解した自然な会話が可能になります。

### 基本的な実装

```ts
import { Memory } from '@mastra/memory';
import { DynamoDBStore } from '@mastra/dynamodb';

// Storageの初期化
const storage = new DynamoDBStore({
  name: 'chat-storage',
  config: {
    tableName: 'chat-memory-table',
    region: 'ap-northeast-1'
  }
});

// Memoryの作成
const memory = new Memory({
  storage: storage,
  options: {
    lastMessages: 20 // 直近20件のメッセージを保持
  }
});
```

### 【重要】 Resource と Thread

Mastraは会話を2つの階層で管理します。

#### Resource（リソース）

誰の記憶か・どこの記憶か等を表します。
- ユーザーID → その人のすべての会話
- チャンネルID → そのチャンネルのすべての会話
- 組織ID → その組織のすべての会話

#### Thread（スレッド）

個別の会話セッションを表します。
- 1つの話題についての一連のやり取り
- 問い合わせ番号やチケット番号に相当


```ts
// パターン1: スレッド独立型
await agent.generate(query, {
  memory: {
    resource: threadId, // 各スレッドが独立
    thread: { id: threadId }
  }
});

// パターン2: リソース横断型
await agent.generate(query, {
  memory: {
    resource: userId, // ユーザー単位で記憶共有
    thread: { id: threadId }
  }
});
```

## 2. Working Memory

Working Memoryは、通常の会話履歴（lastMessages）とは別に管理される、**AIが自動的にユーザーについて覚えておくべき情報を記録する機能**です。例えば「私は田中といいます」と伝えると、AIが自動的にその情報を保存し、次回以降の会話でも「田中さん」と呼びかけられるようになります。

### 具体例

```
User: 私は大阪在住のエンジニアで、Reactが得意です。
AI:   [内部でWorking Memoryを更新: 居住地=大阪、職業=エンジニア、スキル=React]
      承知しました！大阪でエンジニアをされているんですね。

--- 1週間後の別の会話 ---

User: 私に合う勉強会を教えてください。
AI:   大阪でReactの勉強会がありますよ！エンジニアの田中さんにぴったりです。
```

### 設定方法

```ts
const memory = new Memory({
  storage: storage,
  options: {
    lastMessages: 20,
    workingMemory: {
      enabled: true,
      scope: 'resource' // 'thread' または 'resource'
    }
  }
});
```

## 3. セマンティック検索

セマンティック検索（semantic recall）は、**AIが関連する過去の会話を検索して思い出す機能**です。「前に話した〇〇」のような曖昧な言及でも、AIが通常の会話履歴（lastMessages）から外れた、過去の会話から関連する内容を見つけ出します。

### 具体例

```
--- 1ヶ月前の会話（通常の履歴からは外れている）---
User: プロジェクトのデプロイはServerless Frameworkを使います。
AI:   承知しました。

--- 今日の会話 ---
User: デプロイ方法って何でしたっけ？
AI:   [セマンティック検索で「デプロイ」に関連する過去の会話を発見]
      以前お話しいただいたServerless Frameworkを使う方法ですね！
```

通常は直近10〜20件しか参照しませんが、セマンティック検索により**より多くの過去メッセージから関連する内容を瞬時に検索**できます。

### 設定方法

```ts
import { Pinecone } from '@pinecone-database/pinecone';

// Pineconeの初期化
const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY,
  environment: process.env.PINECONE_ENVIRONMENT
});

// MemoryにVector Storeを追加
const memory = new Memory({
  storage: storage,
  vector: pinecone, // Vector Store追加
  options: {
    lastMessages: 20,
    semanticRecall: {
      enabled: true,
      scope: 'resource', // 検索範囲
      topK: 3,           // 上位3件を取得
      messageRange: {    // 各結果の前後文脈
        before: 2,
        after: 2
      }
    }
  }
});
```

### メッセージ取得の優先順位

```typescript
// 取得されるメッセージの構成
const messages = [
  ...現在のスレッドの最新N件, // lastMessages
  ...セマンティック検索結果,  // topK + messageRange
  ...WorkingMemory,
];
```

## 4. プロセッサ（メッセージ最適化）

プロセッサーは、メッセージの前処理・最適化を行うモジュール群です。

### TokenLimiter（トークン数管理）

AIモデルのコンテキストウィンドウ制限に対応します。適切なencodingを使用することで、実際のAPIのトークン計算と一致し、より正確なコンテキスト制限が可能になります。

```typescript
import { TokenLimiter } from '@mastra/memory/processors';

const limiter = new TokenLimiter({
  limit: 8000,
  encoding: o200k_base // GPT-4o用（デフォルト）
});

const memory = new Memory({
  storage: storage,
  processors: [limiter]
});
```

### ToolCallFilter（フィルタリング）

不要なツール呼び出しを履歴から除外します。

```ts
import { ToolCallFilter } from '@mastra/memory/processors';

// 特定のツールを除外
const filterSpecific = new ToolCallFilter({
  exclude: ['debug_tool', 'internal_tool']
});

const memory = new Memory({
  storage: storage,
  processors: [filterSpecific]
});
```

### プロセッサーの組み合わせ

複数のプロセッサを連結して使用することができます。プロセッサは`processors`配列に記載された順番で実行され、直前のプロセッサの出力が次のプロセッサの入力となります。

:::message
**順序は重要です！**

一般的には、**TokenLimiterをチェーンの最後に配置する**のがベストプラクティスです。これにより、他のフィルタリングが行われた後の最終的なメッセージに対して動作し、最も正確なトークン制限の適用が可能になります。
:::

```ts
const memory = new Memory({
  storage: storage,
  processors: [
    new ToolCallFilter({      // 1. ツール除外
      exclude: ['debug_tool']
    }),
    new TokenLimiter(8000),   // 2. トークン制限
  ]
});
```

# 実践的な実装パターン

## パターン1: シンプルなチャットボット

```typescript
// 最小構成（会話履歴の管理のみ）
const memory = new Memory({
  storage: new DynamoDBStore({
    name: 'simple-bot',
    config: {
      tableName: 'simple-bot-memory',
      region: 'ap-northeast-1'
    }
  }),
  options: {
    lastMessages: 30
  }
});

const agent = new Agent({
  name: 'simple-bot',
  model: gpt4,
  memory: memory
});
```

## パターン2: カスタマーサポート

```typescript
// ユーザー横断的な記憶 + Working Memory
const memory = new Memory({
  storage: dynamodbStorage,
  options: {
    lastMessages: 20,
    workingMemory: {
      enabled: true,
      scope: 'resource' // ユーザー単位
    }
  }
});

// ユーザーIDをresourceに設定
await agent.generate(query, {
  memory: {
    resource: userId, // ユーザー横断
    thread: {
      id: ticketId,
    }
  }
});
```

## パターン3: 高度なAIアシスタント

```typescript
// フル機能（Vector Store + Working Memory + プロセッサー）
const memory = new Memory({
  storage: dynamodbStorage,
  vector: pinecone,
  options: {
    lastMessages: 15,
    semanticRecall: {
      enabled: true,
      scope: 'resource',
      topK: 5,
      messageRange: { before: 2, after: 2 }
    },
    workingMemory: {
      enabled: true,
      scope: 'resource'
    }
  },
  processors: [
    new TokenLimiter(8000),
    new ToolCallFilter({
      exclude: ['internal_tool']
    })
  ]
});
```

# 実践例: Slack BotでのMemory活用

実際のSlack botでMastra Memoryを使う場合の実装例を考えてみます。

## Slackの特性を活かした設計

Slackには**スレッド**という概念があるので、これをMastraのThread IDとして活用できます。

```typescript
// Slackイベントハンドラー
async function handleSlackMessage(event: SlackEvent) {
  const threadId = event.thread_ts || event.ts; // Slackスレッドタイムスタンプ
  const userId = event.user; // SlackユーザーID
  const channelId = event.channel; // SlackチャンネルID

  // Memory設定: チャンネル単位でユーザー横断的な記憶を共有
  await agent.generate(event.text, {
    memory: {
      resource: channelId, // チャンネル全体で記憶を共有
      thread: {
        id: threadId,      // Slackスレッドごとに会話を分離
        metadata: {
          userId,          // 投稿者情報を保存
          channelName: await getChannelName(channelId),
        }
      }
    }
  });
}
```

# まとめ

これらの用途に合わせた適切な設定により、文脈を理解し、ユーザーの特性を記憶する賢いAIアシスタントを構築できます。
ほぼ自分用メモとしてまとめたので、何か間違った理解等あればご指摘いただけると幸いです。

# 参考リンク

-   [Mastra公式ドキュメント](https://mastra.ai/)
-   [GitHub - Mastra](https://github.com/mastra-ai/mastra)
