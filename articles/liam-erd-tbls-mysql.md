---
title: "Liam ERDでtblsからサクッとMySQLのER図を作成してみた"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["liamerd", "tbls", "mysql", "er図"]
publication_name: "nextbeat"
published: false
---

## 概要

最近、Liam ERDというOSSのER図自動作成ツールが公開されました。
https://liambx.com/blog/liam-erd-introduction

気になっていたものの、MySQLに対応していなかったので一旦様子見していたのですが、tbls経由でのサポートが発表されたので触ってみます。
https://liambx.com/blog/liam-erd-with-tbls

## Liam ERDとは

OSSのER図自動生成ツールで、SchemaSpyなどに似ていますが、セットアップがより簡単で可読性が高いです。
Web版とCLI版が用意されており、パブリックなGitHubリポジトリであれば、一瞬でER図をレンダリングできます。

## 実行手順

では実際にER図を作成していきます。
今回はサンプルとして、MySQL公式サイトで配布されている `sakila database` という、DVDレンタルショップを模したDBを使用します。

https://dev.mysql.com/doc/index-other.html

ソースファイルはGzip形式やZip形式でダウンロードできるので、こちらを適宜ローカルDBにインポートします。

```sh
# ローカルDBの立ち上げ
docker run --name mysql-local -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql:8.0-oracle

# テーブルを作成
unzip sakila-db.zip
mysql -h 127.0.0.1 -P 3306 -u root -ppassword < sakila-schema.sql
```

### パブリックプロジェクトの場合

パブリックなGitHubリポジトリであれば、スキーマファイルのURLの先頭に `https://liambx.com/erd/p/` をつけるだけでER図をレンダリングできます。

まずtblsを使用してJSON形式のスキーマファイルを生成し、GitHubリポジトリにPushします。

```sh
# tblsをインストール
brew install k1LoW/tap/tbls

# tblsコマンドでJSON形式のスキーマファイルを生成
tbls out --format json mysql://root:password@127.0.0.1:3306/sakila > schema.json
```

リポジトリにPush後は、以下のようにスキーマファイルの先頭に `https://liambx.com/erd/p/` をつけるだけです。

https://liambx.com/erd/p/github.com/taroshun32/liam-erd-tbls-sample/blob/main/schema.json

非常に簡単ですね！
Liam ERDのいいところは、テーブルにマウスオーバーすると関連するテーブルと列が強調表示されるところです。どうしてもテーブルの数が多いとごちゃごちゃしてしまうので、この機能は大変助かります。

![](/images/liam-erd-tbls-mysql/liam-erd.png)

### プライベートプロジェクトの場合

とりあえず試すだけであればURLレンダリングの方法でいいですが、実際に業務で使用するとなるとこのままでは使えません。
Liam ERDはnpmパッケージのCLIツールとしても提供されているので、プライベートプロジェクトの場合はこの方法を利用してER図を作成し、AWS S3やGitHub Pagesなどで静的ホスティングが可能です。

まずLiam CLIを使用してJSON形式のスキーマファイルからER図を作成します。

```sh
$ npx @liam-hq/cli erd build --format tbls --input schema.json

ERD has been generated successfully in the `dist/` directory.
Note: You cannot open this file directly using `file://`.
Please serve the `dist/` directory with an HTTP server and access it via `http://`.
Example:
    $ npx http-server dist/
```

コマンドが正常実行されると、distディレクトリに関連ファイルが生成されます。
ローカル環境で確認する場合は、メッセージにあるように `npx http-server dist/` コマンドで起動し、localhostにアクセスすることでER図を確認できます。

```sh
$ npx http-server dist/

Need to install the following packages:
http-server@14.1.1
...

Available on:
  http://127.0.0.1:8080
  http://192.168.197.0:8080
```

公開したい場合は、以下のようにGitHub Pagesなどでこのディレクトリをホスティングするだけで、チーム内でER図の共有が可能です。

https://taroshun32.github.io/liam-erd-tbls-sample/

## まとめ

Liam ERDを使用することで、手軽にER図を自動生成できることが分かりました。
特にtblsと組み合わせることで、MySQLのスキーマ情報から迅速にER図を作成できます。パブリックなプロジェクトではURLを工夫するだけで簡単に共有でき、プライベートなプロジェクトでもCLIツールを利用して安全にER図を生成・共有できます。
業務でのデータベース設計やチーム内での情報共有にも役立ちそうなので、今後活用していこうと思います。
