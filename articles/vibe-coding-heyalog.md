---
title: "vibe-codingでiOSアプリを1本リリースしてみた話 - モバイル未経験エンジニアの技術選定と反省点"
emoji: "🏠"
type: "idea"
topics: ["expo", "reactnative", "supabase", "revenuecat", "個人開発"]
published: false
---

## はじめに

普段はバックエンドやインフラメインで仕事をしており、モバイルアプリの開発経験はほとんどなかったのですが、AIを活用したvibe-codingが普及してきたことで「自分でもモバイルアプリを作れるのでは」と思い、シンプルに自分が欲しかったアプリを作ってみました。

自分は物をどこにしまったか忘れやすいのですが、頭の中では場所を空間的に把握しているのにリストで管理することにもどかしさを感じ、であれば部屋のレイアウトをそのまま画面に再現して、見た目で直感的に管理できるアプリを作ってしまおう、と考えたのがきっかけです。
## アプリの概要

今回開発した **「へやログ」** は、部屋の間取りをビジュアルで再現し、家具をタップしてモノを登録・管理するiOSアプリです。

![](/images/vibe-coding-heyalog/heyalog-aso.png)

主な機能は3つです。

- 間取りをグリッドで再現し、家具を自由に配置する
- 家具をタップして持ち物を登録する
- キーワードで検索して選択すると、該当の家具がハイライトされる

「リスト型ではなく、空間で直感的に管理する」というコンセプトが差別化のポイントです。

## 技術スタック

| カテゴリ | 技術 |
|---|---|
| フレームワーク | Expo SDK 54 |
| 言語 | TypeScript |
| ルーティング | Expo Router v6 |
| 状態管理 | Zustand v5 |
| アニメーション | Reanimated v4 |
| DB / 認証 | Supabase (PostgreSQL + 匿名認証) |
| 課金 | RevenueCat v9 |
| 間取り描画 | react-native-svg |

## インフラ・サービス選定の考え方

個人アプリを作るにあたって、とにかく **「サーバー管理コストをゼロにする」** というところにはこだわりました。
運用に費用をかけたくないというのは正直なところで、インフラやサービスの選定はすべて「無料枠で個人アプリとして成立するか」を基準にしています。

### Supabase

データベースと認証には Supabase を採用しました。

https://supabase.com/

PostgreSQL をフルマネージドで提供しており、無料枠でも個人アプリとして十分な規模まで使えます。インフラ管理が不要で、RLS（Row Level Security）によるアクセス制御もシンプルに記述できます。

認証は匿名認証から始め、あとからメールを紐付ける方式にしました。「アプリを開いたらすぐ使える」という体験を優先しつつ、機種変更時のデータ引き継ぎにも対応しています。

また、将来的に家族共有機能を実装したいと考えています。その際にユーザー間のデータ共有やリアルタイム同期が必要になりますが、Supabase であれば既存の延長線上で実装できる点も選定理由のひとつです。

唯一サーバーサイドの処理が必要だったのがアカウント削除です。`auth.admin.deleteUser()` は `service_role` キーが必要なサーバー専用APIのため、クライアントから直接呼ぶことができません。ここでは Supabase Edge Functions を使い、Deno ランタイム上で管理者権限による削除処理を実行しています。

```typescript
// Edge Function
// ユーザーのJWTで本人確認 → service_role の管理者クライアントで削除
const supabaseAdmin = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
);
await supabaseAdmin.auth.admin.deleteUser(user.id);
// ON DELETE CASCADE により関連データ（部屋・家具・アイテム）も全て自動削除
```

クライアント側は `supabase.functions.invoke('delete-account')` の1行で呼び出せます。Edge Functions も無料枠で十分運用でき、デプロイは `supabase functions deploy` コマンドひとつで完了するため、個人開発でも気軽に使える仕組みでした。

### RevenueCat

課金管理には RevenueCat を採用しました。

https://www.revenuecat.com/

Supabase と同様に、一定の収益規模に達するまでは無料で利用できます。App Store の In-App Purchase を直接扱うのは手続きが複雑になりがちですが、RevenueCat を挟むことで「パッケージの取得 → 購入 → エンタイトルメントの確認」という流れをシンプルなAPIで実装できました。

月額・年額・買い切りの3プランを用意しており、offerings の取得から UI への反映まで以下のような形で書けます。

```typescript
const offerings = await Purchases.getOfferings();
const packages = offerings.current?.availablePackages ?? [];
// $rc_monthly / $rc_annual / $rc_lifetime でフィルタして表示
```

PRO 判定はクライアント側の `entitlements.active` で行っており、Supabase 側に premium カラムは持たせていません。RevenueCat の appUserID には Supabase の user.id をそのまま使うことで、ログイン・ログアウト時の同期も自然な形で実装できています。

## vibe-codingについて

モバイルアプリの開発経験がほぼない状態で作り始めたため、AIをフル活用しています。Reanimated を使ったドラッグ実装や、iOS の `CFStringTransform` を利用した漢字のひらがな読み変換など、個人では相当時間がかかっていたであろう部分を vibe-coding では秒で実装することができました。
React Native 特有のレンダリングの仕組みやジェスチャーの扱いは、実際に動かしながら理解していくことが多かったです。

## vibe-codingでの反省点

ダークモード対応では、`lightColors` / `darkColors` の2パレットを定義し、Zustand + カスタム `useColors()` hook でテーマに応じたカラーを返す構成にしました。動作自体は問題なく、設定画面からシステム / ライト / ダークの3択で切り替えられます。

ただ、あとから調べてみると、Expo Router / React Navigation には `ThemeProvider` + `useTheme()` というテーマ基盤が標準で用意されており、本来はそちらに乗るのが推奨パターンのようです。現状はフレームワーク標準のテーマ基盤と独自hookの二重管理になっており、設計としてはあまりよろしくありません。

また、オフライン対応にも反省点があり、Supabase を使いたいという気持ちが先行して、全データをリモートに保存する前提で設計してしまいました。現状、ネットワーク接続がないとアプリがほぼ使えない状態です。持ち物管理アプリは日常的に使うものなので、電波の悪い場所でもストレスなく動くべきでした。

Supabase との親和性が高いオフライン同期ソリューションとして PowerSync が気になっています。ローカルに SQLite を持ちつつ Postgres と自動同期する仕組みで、Supabase Auth との統合もネイティブでサポートされているため、既存の構成に後付けしやすそうです。今後のアップデートで対応を検討しています。

https://www.powersync.com/

vibe-coding で進めていると「動いたからOK」で先に進みがちで、フレームワークが提供する仕組みやアプリとしての基本要件を事前に把握せず進めてしまうのは、あるあるな落とし穴だと感じました。

## おわりに

モバイルアプリの世界はほとんど知らない状態からのスタートでしたが、この開発を通じて App Store のリリースフロー、In-App Purchase の仕組み、React Native のレンダリングやジェスチャーの扱いなど、これまで触れる機会のなかった領域を一通り経験できました。vibe-coding のおかげで「とりあえず動くものを作って学ぶ」サイクルを高速に回せたのは、いい機会だったと思います。

Supabase と RevenueCat を組み合わせることで、サーバー管理コストをかけずに認証・DB・課金の基盤を整えられたのは、個人開発という観点で合理的な選択でした。同じような構成を検討している方の参考になれば幸いです。

*App Store にて「へやログ」で検索いただくか、以下のリンクからご覧いただけます。*

{ここにApp Storeリンクまたはバッジ画像}