---
title: "リレーション定義のないDBからも自動でER図を作成したい！（SchemaSpyを応用してみる）"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["db", "mysql", "schemaspy", "er図", "shell"]
published: false
---

## 概要

DBのスキーマ定義やER図の運用は、可能な限り手動ではなく自動化したいですよね。
そのような場合に便利なのがSchemaSpyというツールで、稼働中のDBに接続してテーブル構造を解析し、以下のようなER図などのHTMLドキュメントを自動で作成してくれます。

https://schemaspy.org/

![](/images/schemaspy-auto-relation/er.png)

非常に便利なので私自身も使用しているのですが、当然リレーションが定義されていない場合ER図は作成できません。
追加でXMLファイルを作成することにより、リレーションやその他のメタ情報を追加できるのですが、これは手動で更新する必要があり、メンテナンス漏れが発生する可能性があります。

そこで今回はちょっと工夫して、カラムのコメントからXMLファイルを自動生成してER図を運用する方法を考えてみたので、紹介したいと思います。
(※この方法は半分遊びのようなものなので、気軽に読んでみてくださいw)

サンプルコードは以下のリポジトリにあります。

https://github.com/taroshun32/schemaspy-metadata-generator

## SchemaSpy の使い方

SchemaSpyをご存知ない方のために、まずは基本的な使用方法を説明します。
dockerベースとjarベース(Java製アプリケーション)の2種類で使用することができるのですが、今回はdockerベースを使用します。

### DBの準備

MySQL公式サイトで配布されているサンプルの `world database` がシンプルで使いやすそうなため、今回はこちらを使用します。

https://dev.mysql.com/doc/index-other.html

ソースファイルはGzip形式やZip形式でダウンロードできるので、こちらを適宜ローカルDBにインポートします。（今回はあえてダウンロード後にリレーション定義を削除しましょう）
:::details world.sql
https://github.com/taroshun32/schemaspy-metadata-generator/blob/main/world.sql
:::

M1Macに対応できるよう、MySQLについてはoracleイメージを採用しています。

```sh
# ローカルDBの立ち上げ
docker run --name mysql-local -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql:8.0-oracle

# テーブルを作成
unzip world.sql.zip
mysql -h 127.0.0.1 -P 3306 -u root -ppassword < world.sql
```

### ドキュメントを生成

SchemaSpyをベースに、MySQL8.0とM1Macに対応したDockerfileを作成します。

```dockerfile:Dockerfile
FROM --platform=linux/amd64 schemaspy/schemaspy:6.1.0

ENV TZ=Asia/Tokyo

USER root

COPY schemaspy.properties .

ADD https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.30/mysql-connector-java-8.0.30.jar \
    /drivers/mysql-connector-java-8.0.30.jar
```

DBの接続情報を記載したschemaspy.propertiesファイルを作成します。
Dockerコンテナの中から、ホスト側で動作しているサービスに接続するには、IPアドレスの代わりに特殊なDNS名 `host.docker.internal` を使用します。（`localhost` だとコンテナ自身を参照してしまうのでうまくいきません）

```properties:schemaspy.properties
schemaspy.t = mysql

schemaspy.dp   = /drivers/mysql-connector-java-8.0.30.jar
schemaspy.host = host.docker.internal
schemaspy.port = 3306
schemaspy.db   = world
schemaspy.u    = root
schemaspy.p    = password

schemaspy.implied = true
schemaspy.norows  = true
schemaspy.nopages = true

schemaspy.s = world
```

これで準備は整ったので、実際にDockerfileをビルドして実行してみましょう。

```sh
docker build -t my-schemaspy .
docker run --name my-schemaspy --rm -v $PWD/schema:/output my-schemaspy
```

するとschemaディレクトリ配下にHTMLドキュメントが生成されているかと思います。
しかし確認してみるとER図がうまく生成されていないですよね。
(以下のような状態になるかと思います)

![](/images/schemaspy-auto-relation/er2.png)

もちろんこれは今リレーションを定義していないので当然です。
ではここからリレーション情報をSchemaSpyにメタデータとして渡す方法を見ていきます。

## XMLファイルでのリレーション情報追加

SchemaSpyでは、追加でXMLファイルを作成することにより、リレーションやその他のメタ情報の追加が可能です。

https://schemaspy.readthedocs.io/en/v6.0.0/configuration.html#add-relationships

今回の場合は以下のようにXMLファイルを作成します。

```xml:metadata.xml
<schemaMeta xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://schemaspy.org/xsd/6/schemameta.xsd" >
    <tables>
        <table name="city">
            <column name="CountryCode">
                <foreignKey table="country" column="code" />
            </column>
        </table>
        <table name="countrylanguage">
            <column name="CountryCode">
                <foreignKey table="country" column="code" />
            </column>
        </table>
    </tables>
</schemaMeta>
```

この状態でDockerfileとschemaspy.propertiesファイルにmetadata.xmlを参照するよう追記し、再度SchemaSpyを実行してみると、今度はER図が作成されるかと思います。

```diff dockerfile:Dockerfile
  COPY schemaspy.properties .
+ COPY metadata.xml .
```

```diff properties:schemaspy.properties
  schemaspy.s    = world
+ schemaspy.meta = metadata.xml
```

![](/images/schemaspy-auto-relation/er3.png)

## XMLファイルを自動生成

ここまでの手順で、XMLファイルを作成すればリレーションを定義していなくてもER図を作成することが可能だとわかりましたが、Schema定義に追加や変更が入るたびにこのXMLファイルをメンテナンスしなきゃいけないですよね...
できればSchema定義が変更されても、何も手動メンテナンスなしでER図が最新の状態に更新されて欲しいものです。

そこで今回はカラムのコメントに `foreignKey` の情報を記載し、そこからXMLファイルを自動生成してみます。

まずは `city` テーブルと`countrylanguage` テーブルの `CountryCode` カラムに以下のフォーマットでコメントを設定します。
`fk:{テーブル名}.{カラム名}`

```diff sql:world.sql
- CountryCode` char(3) NOT NULL DEFAULT '',
+ CountryCode` char(3) NOT NULL DEFAULT '' COMMENT 'fk:country.code',
```

次に実際にDBに接続して全テーブルのカラムのコメントを参照し、XMLファイルを作成するシェルスクリプトを作成します。
(ここは泥臭くゴリゴリ書いてますw)

```sh:create_meta.sh
#!/bin/bash

# データベース接続設定
USER="root"
PASSWORD="password"
DATABASE="world"
HOST="127.0.0.1"

# metadata.xmlファイルを初期化
echo '<schemaMeta xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://schemaspy.org/xsd/6/schemameta.xsd" >' > metadata.xml
echo '    <tables>' >> metadata.xml

# SQLクエリを実行し、結果を行ごとに処理
mysql -h $HOST -u $USER -p$PASSWORD -D $DATABASE -sN -e \
"SELECT TABLE_NAME, COLUMN_NAME, COLUMN_COMMENT FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA = '$DATABASE' AND COLUMN_COMMENT LIKE 'fk:%'" | \
while read TABLE_NAME COLUMN_NAME COLUMN_COMMENT
do
    # コメントから外部キーテーブルとカラムを抽出
    FK_TABLE=$(echo $COLUMN_COMMENT | cut -d':' -f2 | cut -d'.' -f1)
    FK_COLUMN=$(echo $COLUMN_COMMENT | cut -d':' -f2 | cut -d'.' -f2)

    # XMLファイルにリレーション情報を追加
    echo "        <table name=\"$TABLE_NAME\">" >> metadata.xml
    echo "            <column name=\"$COLUMN_NAME\">" >> metadata.xml
    echo "                <foreignKey table=\"$FK_TABLE\" column=\"$FK_COLUMN\" />" >> metadata.xml
    echo "            </column>" >> metadata.xml
    echo "        </table>" >> metadata.xml
done

# XMLファイルを閉じる
echo '    </tables>' >> metadata.xml
echo '</schemaMeta>' >> metadata.xml
```

これでXMLファイルを自動生成する準備ができました。
なんと、Dockerfileのビルド前にこの `create_meta.sh` を実行するだけで、ER図が最新の状態に更新されるはずです。（やったね！）

## まとめ

今回はリレーション定義のないDBから自動でER図を作成する方法を解説してみました。
思いつきで作ってみたので泥臭さがすごいですが、コメントさえつけ忘れなければドキュメントの手動メンテナンスの必要がなくなるのは便利なのではないでしょうか。

もっといい方法あるよーという方いたらぜひコメントで教えてください〜。
最後まで読んでいただきありがとうございました。

## 参考

- [M1Mac × Docker × SchemaSpy × MySQL8.0でテーブル定義書とER図を自動生成してみる](https://gmor-sys.com/2022/10/19/db-document-autocreation-tool/)
