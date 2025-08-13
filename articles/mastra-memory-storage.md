---
title: "MastraのMemory機能を深掘る"
emoji: "💬"
type: "tech"
topics: ["mastra", "ai", "aiエージェント", "nodejs", "dynamodb"]
published: false
---


# 概要

AIチャットボットやエージェントを構築する際、最も重要な要素の一つが「記憶」です。ユーザーとの過去の会話を記憶し、文脈を理解した上で継続利用するためには、適切なメモリシステムが不可欠です。

筆者も最近、社内AIエージェント構築を行う機会が増えていますが、毎回メモリシステムの構築に苦戦しています。そこで今回は、MastraのMemory機能を詳しく深掘りし、実践的な実装パターンまで含めて理解を深める目的でまとめてみます。

https://mastra.ai/ja/docs/memory/overview

:::message
本記事は以下のバージョンに基づいて執筆しています。
- `@mastra/memory`: v0.12.1
- `@mastra/core`: v0.13.1
  :::

# Mastra Memoryとは

Mastra Memoryは、AIエージェントの会話管理に特化した包括的なライブラリです。以下の4つの主要機能を提供します。

1.  **会話履歴の管理**  - メッセージの保存・取得・管理
2.  **Working Memory**  - スレッド横断的な永続記憶
3.  **Semantic Recall**  - 意味的に関連する過去の会話を検索
4.  **メモリプロセッサ**  - メッセージの前処理・最適化

# アーキテクチャ

Mastra Memoryは3層のシンプルな構造で、各層が明確に分離されています。

```
┌─────────────────────┐
│       Memory        │ ← 統一されたAPI、4つの主要機能を提供
├─────────────────────┤
│      Storage        │ ← 会話データの永続化（必須）
├─────────────────────┤
│    Vector Store     │ ← セマンティック検索用（オプション）
└─────────────────────┘
```

## 各層の責務

### Memory層

開発者が直接触れる唯一のインターフェース。Storage/Vector Storeの違いを意識せずに、統一されたAPIで全機能を利用可能。

### Storage層（必須）

メッセージとメタデータを永続化。複数のアダプターから選択可能。

-   `@mastra/dynamodb`  - AWS DynamoDB
-   `@mastra/postgres`  - PostgreSQL
-   `@mastra/upstash`  - Upstash Redis
-   `@mastra/libsql`  - LibSQL
- その他多数のアダプターに対応

### Vector Store層（オプション）

Semantic Recall機能を使う場合のみ必要。古い会話履歴をセマンティック検索するためのベクトルデータベース。

# 各主要機能の詳細

## 1. 会話履歴の管理

会話履歴の管理は、Mastra Memoryの最も基本的な機能です。過去のメッセージを自動的に保存し、新しいリクエスト時に**現在のThread内から**適切な数の履歴（デフォルト10件）をコンテキストに含めます。これにより、AIが文脈を理解した自然な会話が可能になります。

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

### Resource と Thread

Mastraでは、効率的な記憶管理のために会話を2つの階層で整理します。

#### Resource

「誰の・どこの記憶か」を識別する概念です。Working MemoryやSemantic Recallのスコープを決定します。

#### Thread

「どの会話か」を識別する概念です。1つのトピックに関する一連のメッセージをグループ化します。`lastMessages`はこのThread内のメッセージのみを取得します。

:::message
Thread IDは全Resource間でグローバルに一意である必要があります。同じThread IDを異なるResourceで使い回すことはできません。
:::

```ts
// パターン1: スレッド独立型
await agent.generate(query, {
  memory: {
    resource: threadId, // 各スレッドが独立(resourceとthreadが同じ値)
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

Working Memoryは、**AIが会話を通じて学んだ重要情報を永続的に記録する機能**です。通常の会話履歴（lastMessages）とは別に管理され、AIが自動的に`updateWorkingMemory`ツールを呼び出して更新します。
スコープ設定により、スレッド単位またはリソース横断的に記憶を保持できます。

### 動作の仕組み

```
ユーザー: 「私の名前は田中です。大阪在住で、Reactが得意です。」
    ↓
AIの内部処理:
  1. 情報を認識
  2. updateWorkingMemoryツールを呼び出し
  3. テンプレートに従って情報を保存
    ↓
AIの応答: 「田中さん、大阪でReactを使われているんですね！」

--- 数日後の別の会話 ---

ユーザー: 「私に合う勉強会を教えてください。」
    ↓
AIの内部処理:
  1. Working Memoryがシステムメッセージとして自動的に提供される
  2. 田中さん、大阪在住、React得意という情報を確認
  3. その情報を基に回答を生成
    ↓
AIの応答: 「大阪でReactの勉強会がありますよ！田中さんにぴったりです。」
```

### 設定方法

Working Memoryのテンプレートは自由にカスタマイズ可能です。デフォルトでは基本的なユーザー情報を記録しますが、用途に応じて項目を追加・変更できます。

```ts
const memory = new Memory({
  storage: storage,
  options: {
    workingMemory: {
      enabled: true,
      scope: 'resource', // 'thread' または 'resource'
      // テンプレートをカスタマイズ
      template: `
# ユーザー情報
- **名前**: 
- **居住地**: 
- **職業**: 
- **興味・関心**: 
`
    }
  }
});
```

### JSONスキーマによる構造化

より厳密なデータ管理が必要な場合、ZodまたはJSONSchemaを使用できます。

```ts
import { z } from 'zod';

const userProfileSchema = z.object({
  name: z.string().optional(),
  location: z.string().optional(),
  preferences: z.object({
    language: z.string().optional(),
    timezone: z.string().optional(),
  }).optional(),
});

const memory = new Memory({
  storage: storage,
  options: {
    workingMemory: {
      enabled: true,
      schema: userProfileSchema // Zodスキーマを指定
    }
  }
});
```

## 3. Semantic Recall

Semantic Recallは、**セマンティック検索を使って関連する過去の会話を自動的に思い出す機能**です。通常の会話履歴（lastMessages）から外れた古いメッセージでも、意味的に関連する内容を検索してAIのコンテキストに含めます。

:::message alert
Semantic Recallを使用するには、Storageとは別に以下の2つが必須です。
1. Vector Store - セマンティック検索
2. Embedder - テキストの埋め込み
:::

### 動作の流れ

```
1. ユーザーが質問
   ↓
2. 質問をベクトル化（埋め込み）
   ↓
3. Vector Storeで類似検索
   ↓
4. 関連する過去のメッセージを取得
   ↓
5. AIのコンテキストに追加して応答生成
```

### 設定方法

```ts
import { Memory } from '@mastra/memory';
import { Pinecone } from '@pinecone-database/pinecone';
import { fastembed } from '@mastra/fastembed';

const memory = new Memory({
  storage: storage,     // メッセージ保存用
  vector: pinecone,     // セマンティック検索用（必須）
  embedder: fastembed,  // 埋め込みモデル（必須）
  options: {
    lastMessages: 20,
    semanticRecall: {
      enabled: true,
      topK: 5,         // 取得する類似メッセージ数
      messageRange: {  // 各結果の前後文脈
        before: 2,
        after: 2
      },
      scope: 'resource'  // 'thread' または 'resource'
    }
  }
});
```

## 4. メモリプロセッサ

メモリプロセッサは、メッセージの前処理・最適化を行うモジュール群です。

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

### プロセッサの組み合わせ

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
// フル機能（Vector Store + Working Memory + プロセッサ）
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
Slackには**スレッド**という概念があるので、これをMastraのThread IDとして活用できます。

```typescript
// Memory設定（チャンネル単位でWorking Memory共有）
const memory = new Memory({
  storage: dynamodbStorage,
  options: {
    lastMessages: 30,   // 各スレッド内の履歴
    workingMemory: {
      enabled: true,
      scope: 'resource' // チャンネル全体で共有
    }
  }
});

const agent = new Agent({
  name: 'slack-bot',
  model: gpt4o,
  memory: memory
});

// Slackイベントハンドラー
async function handleSlackMessage(event: SlackEvent) {
  const threadId = event.thread_ts; // Slackスレッドタイムスタンプ
  const userId = event.user;        // SlackユーザーID
  const channelId = event.channel;  // SlackチャンネルID

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

この設定により、以下が可能となります。
- **各Slackスレッド内**: 最新30件の会話履歴を保持
- **Slackチャンネル全体**: プロジェクト情報、決定事項などをWorking Memoryで共有

# まとめ

これらの機能を用途に合わせて適切に設定することで、文脈を理解し、ユーザーの特性を記憶する賢いAIアシスタントを構築できます。何か間違った理解等あればご指摘いただけると幸いです。

# 参考リンク

-   [Mastra公式ドキュメント](https://mastra.ai/)
-   [GitHub - Mastra](https://github.com/mastra-ai/mastra)
