---
title: "[コスト削減] RDSのログをS3にダイレクトで自動転送する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["コスト削減", "AWS", "RDS", "LOG", "S3"]
publication_name: "nextbeat"
published: false
---

## 概要

RDSのログファイルは、デフォルトでは一定期間後に削除されます。
https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.MySQL.LogFileSize.html

監査やトラブル対応のために長期保存する場合、CloudWatch Logsを経由してS3に保存する方法を採用するケースが一般的だと思いますが、この方法ではデータ転送で中々のコストが発生します。

このコスト削減を目的として、RDSのログファイルを直接S3へ転送する自動化スクリプトをPythonで作成してみたので、自分への備忘録として残しておきます。

## アーキテクチャ

今回実装したシステムのアーキテクチャは以下です。

1. AWS EventBridgeで定期的にLambda関数をトリガー
2. Lambda関数がRDSクラスターの各インスタンスからログファイルをダウンロード
3. ダウンロードしたログファイルを圧縮し、S3にアップロード

ログファイルは以下のような構造でS3バケットに保存されます。

```
s3://{バケット名}/
    ├── {クラスター名}/
    │   └── audit/
    │       └── {年}/
    │           └── {月}/
    │               └── {日}/
    │                   └── {時}/
    │                       └── {インスタンス名}/
    │                           └── {ログファイル名}.gz
    └── {別のクラスター名}/
        └── ...
```

例えば、`my-rds-cluster`というクラスター名で、`instance-1`というインスタンス名の場合、実際のS3オブジェクトキーは以下のようになります。

```
my-rds-cluster/audit/2024/10/01/12/instance-1/audit.log.2024-10-01-12-00.0.gz
```

## サンプルコード

https://github.com/taroshun32/aurora-mysql-log-archive
SAMを使用してLambda関数で作成してます。今回は実装コードの概要だけ簡単に解説します。

## 実装の詳細

完全なコードは以下です。
:::details スクリプト
```python
import boto3
import gzip
import urllib.request
import os
from io import BytesIO
from datetime import datetime, timedelta
from botocore.awsrequest import AWSRequest
import botocore.auth as auth

def main(event, context):
    # 定数
    S3_BUCKET = os.getenv('S3_BUCKET')
    CLUSTERS  = os.getenv('CLUSTERS').split(',')

    # クライアント
    rds_client = boto3.client('rds')
    s3_client  = boto3.client('s3')

    # セッション情報
    session     = boto3.session.Session()
    region      = session.region_name
    credentials = session.get_credentials()

    # SigV4署名
    sigv4auth = auth.SigV4Auth(credentials, 'rds', region)

    # 最終更新時刻が2時間15分前から1時間前までのログを処理対象とする
    now           = datetime.now()
    two_hours_ago = int((now - timedelta(hours=2, minutes=15)).timestamp() * 1000)
    one_hour_ago  = int((now - timedelta(hours=1)).timestamp() * 1000)

    # クラスター一覧を取得し、ループ処理
    clusters = rds_client.describe_db_clusters(
        Filters=[{
            'Name':   'db-cluster-id',
            'Values': CLUSTERS
        }]
    )
    for cluster in clusters['DBClusters']:
        cluster_name = cluster['DBClusterIdentifier']

        print(f"Processing cluster: {cluster_name}")

        # インスタンス一覧を取得し、ループ処理
        instances = cluster['DBClusterMembers']
        for instance in instances:
            instance_name = instance['DBInstanceIdentifier']
            print(f"Processing instance: {instance_name}")

            # 対象のログファイル一覧を取得
            download_log_file_names = []
            marker = None
            while True:
                params = {
                    'DBInstanceIdentifier': instance_name,
                    'FileLastWritten':      two_hours_ago,
                    'MaxRecords':           256
                }
                if marker:
                    params['Marker'] = marker

                logs = rds_client.describe_db_log_files(**params)

                download_log_file_names.extend([
                    log['LogFileName'] for log in logs['DescribeDBLogFiles']
                    if two_hours_ago <= log['LastWritten'] < one_hour_ago and log['LogFileName'].startswith('audit/')
                ])

                marker = logs.get('Marker')
                if not marker:
                    break

            # ログファイルをループ処理
            for log_file_name in download_log_file_names:
                print(f"Processing log file: {log_file_name}")

                # S3オブジェクトキーを生成
                timestamp  = datetime.strptime(log_file_name.split('.')[3], '%Y-%m-%d-%H-%M')
                object_key = f"{cluster_name}/audit/{timestamp.year}/{timestamp.month:02}/{timestamp.day:02}/{timestamp.hour:02}/{instance_name}/{log_file_name.split('/')[-1]}.gz"

                # S3で同一のファイル名が存在するか確認
                try:
                    s3_client.head_object(Bucket=S3_BUCKET, Key=object_key)
                    print(f"File already exists in S3: {object_key}. Skipped.")
                    continue  # ファイルが存在する場合は、処理をスキップ
                except s3_client.exceptions.ClientError as e:
                    pass # ファイルが存在しない場合は、処理を続行

                # 署名付きURLを生成
                host   = f"rds.{region}.amazonaws.com"
                url    = f"https://{host}/v13/downloadCompleteLogFile/{instance_name}/{log_file_name}"
                awsreq = AWSRequest(method='GET', url=url)
                sigv4auth.add_auth(awsreq)

                req = urllib.request.Request(url, headers={
                    'Authorization':        awsreq.headers['Authorization'],
                    'Host':                 host,
                    'X-Amz-Date':           awsreq.context['timestamp'],
                    'X-Amz-Security-Token': credentials.token
                })

                # ログファイルをダウンロード
                with urllib.request.urlopen(req) as response:
                    log_data = response.read()

                # ファイルが空の場合は除外
                if len(log_data) == 0:
                    print(f"Log file {log_file_name} is empty. Skipping.")
                    continue

                # ログファイルを圧縮
                compressed_data = BytesIO()
                with gzip.GzipFile(fileobj=compressed_data, mode='wb') as f:
                    f.write(log_data)
                compressed_data.seek(0)

                # S3にアップロード
                s3_client.upload_fileobj(
                    Fileobj=compressed_data,
                    Bucket=S3_BUCKET,
                    Key=object_key
                )

    return {
        'statusCode': 200,
        'body': 'Audit log file processing completed.'
    }
```
:::

各クラスター内のインスタンス情報を取得して二重ループを開始し、全てのインスタンスを対象に以下の流れで処理を行います。

1. audit-logファイルの一覧取得
2. S3オブジェクトキーの生成と重複チェック
3. ログファイルのダウンロード
4. ログファイルの圧縮とS3へのアップロード

順番に解説します。

### 1. audit-logファイルの一覧取得

RDS APIを使用して直近更新されたaudit-logファイル名の一覧を取得します。
また、指定した時間範囲内に更新されたファイルをフィルタリングします。

```python
# 対象のログファイル一覧を取得
download_log_file_names = []
marker = None
while True:
    params = {
        'DBInstanceIdentifier': instance_name,
        'FileLastWritten':      two_hours_ago,
        'MaxRecords':           256
    }
    if marker:
        params['Marker'] = marker

    logs = rds_client.describe_db_log_files(**params)

    download_log_file_names.extend([
        log['LogFileName'] for log in logs['DescribeDBLogFiles']
        if two_hours_ago <= log['LastWritten'] < one_hour_ago and log['LogFileName'].startswith('audit/')
    ])

    marker = logs.get('Marker')
    if not marker:
        break

...
```

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds/client/describe_db_log_files.html

### 2. S3オブジェクトキーの生成と重複チェック

各ログファイルに対して、S3上でのユニークなオブジェクトキーを生成します。
S3上に同名のファイルが既に存在するかチェックし、存在する場合は処理をスキップします。

```python
# S3オブジェクトキーを生成
timestamp  = datetime.strptime(log_file_name.split('.')[3], '%Y-%m-%d-%H-%M')
object_key = f"{cluster_name}/audit/{timestamp.year}/{timestamp.month:02}/{timestamp.day:02}/{timestamp.hour:02}/{instance_name}/{log_file_name.split('/')[-1]}.gz"

# S3で同一のファイル名が存在するか確認
try:
    s3_client.head_object(Bucket=S3_BUCKET, Key=object_key)
    print(f"File already exists in S3: {object_key}. Skipped.")
    continue  # ファイルが存在する場合は、処理をスキップ
except s3_client.exceptions.ClientError as e:
    pass # ファイルが存在しない場合は、処理を続行

...
```

### 3. ログファイルのダウンロード

RDSからログファイルをダウンロードするために、署名付きURLを生成します。
生成したURLを使用してログファイルをダウンロードします。

```python
# 署名付きURLを生成
host   = f"rds.{region}.amazonaws.com"
url    = f"https://{host}/v13/downloadCompleteLogFile/{instance_name}/{log_file_name}"
awsreq = AWSRequest(method='GET', url=url)
sigv4auth.add_auth(awsreq)

req = urllib.request.Request(url, headers={
    'Authorization':        awsreq.headers['Authorization'],
    'Host':                 host,
    'X-Amz-Date':           awsreq.context['timestamp'],
    'X-Amz-Security-Token': credentials.token
})

# ログファイルをダウンロード
with urllib.request.urlopen(req) as response:
    log_data = response.read()

# ファイルが空の場合は除外
if len(log_data) == 0:
    print(f"Log file {log_file_name} is empty. Skipping.")
    continue
```

今回使用したファイルダウンロード用APIの詳細に関しては以下を参照してください。
https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/DownloadCompleteDBLogFile.html

### 4. ログファイルの圧縮とS3へのアップロード

ダウンロードしたログファイルをgzip形式で圧縮し、S3にアップロードします。

```python
# ログファイルを圧縮
compressed_data = BytesIO()
with gzip.GzipFile(fileobj=compressed_data, mode='wb') as f:
    f.write(log_data)
compressed_data.seek(0)

# S3にアップロード
s3_client.upload_fileobj(
    Fileobj=compressed_data,
    Bucket=S3_BUCKET,
    Key=object_key
)
```

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/upload_fileobj.html

## まとめ

今回の実装により、RDSのログファイルを直接S3に転送することができ、データ転送コストを大幅に削減することができます。
また、Athenaを使用することで大量のログデータを効率的に分析し、セキュリティ監査やパフォーマンス最適化のための洞察を得ることもできます。

AWSには他にもコスト削減の余地があるサービスは多くあると思うので、サービスを最大限に活用しつつ、コスト効率の良いアーキテクチャ設計を進めていきたいと思います。
