# Private EC2 のログを CloudWatch Logs に送信する構成

## 概要

Private Subnet に配置した EC2 インスタンスから、テスト用ログファイルを CloudWatch Logs に送信する構成を作成した。

EC2 には Public IP を付与せず、SSH ポートも開放していない。

EC2 への接続には Systems Manager Session Manager を使用した。
また、CloudWatch Agent のインストールには Systems Manager Run Command を使用した。

## 今回の目的

* Private EC2 に Session Manager で接続する
* EC2 内にテスト用ログファイルを作成する
* CloudWatch Agent をインストールする
* CloudWatch Agent の設定ファイルを作成する
* EC2 内のログを CloudWatch Logs に送信する
* CloudWatch Logs の画面でログを確認する

## 構成

```text
AWS Cloud
└── VPC
    └── Private Subnet
        └── EC2
            ├── SSM Agent
            ├── CloudWatch Agent
            └── /var/log/toki-test.log
                    ↓
              CloudWatch Logs
```

Private EC2 はインターネットへ直接接続しない。

AWS サービスとの通信には VPC Endpoint を使用した。

```text
Private EC2
├── SSM 系 VPC Endpoint
│   ├── ssm
│   ├── ssmmessages
│   └── ec2messages
│
├── S3 Gateway Endpoint
│   └── CloudWatch Agent のパッケージ取得
│
└── CloudWatch Logs Endpoint
    └── ログ送信
```

## 作成した主なリソース

### ネットワーク

```text
VPC
Private Subnet
Private Subnet 用 Route Table
```

### VPC Endpoint

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
com.amazonaws.ap-northeast-1.logs
com.amazonaws.ap-northeast-1.s3
```

### IAM Role に付与したポリシー

```text
AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy
```

### EC2 内で作成したログ

```text
/var/log/toki-test.log
```

### CloudWatch Logs のロググループ

```text
/aws/ec2/toki-test-log
```

## 実施結果

EC2 内で次のようなログを追記した。

```text
cloudwatch logs test after agent start ...
```

その後、CloudWatch Logs の画面で以下を確認できた。

```text
ロググループ
└── /aws/ec2/toki-test-log
    └── EC2 のインスタンス ID
        └── EC2 内で追記したログ
```

これにより、Private EC2 内のログを CloudWatch Logs に集約できることを確認した。

## 今回の学び

CloudWatch Logs が EC2 内のログを直接見に来るわけではない。

EC2 内で動作する CloudWatch Agent がログファイルを読み取り、CloudWatch Logs へ送信する。

```text
EC2 内のログファイル
↓
CloudWatch Agent
↓
CloudWatch Logs
```

また、Private EC2 ではインターネットへ直接接続できないため、用途ごとに VPC Endpoint が必要になる。

## ドキュメント

* [構築手順](docs/setup.md)
* [役割の整理・トラブルシューティング](docs/notes-and-troubleshooting.md)

