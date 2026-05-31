# Private EC2 から S3 Gateway Endpoint 経由で S3 にアクセスする

## 概要

Private Subnet に配置した EC2 インスタンスから、インターネットを経由せずに Amazon S3 へアクセスする構成を作成した。

EC2 には Public IP を付与せず、SSH ポートも開放していない。
EC2 への接続には AWS Systems Manager Session Manager を使用した。

また、EC2 から S3 へのアクセスには S3 Gateway Endpoint を使用した。

```text
自分のPC
↓
AWS Systems Manager Session Manager
↓
SSM系 Interface Endpoint
↓
Private EC2
↓
S3 Gateway Endpoint
↓
Amazon S3
```

今回の実践では、S3 から EC2 へのダウンロードと、EC2 から S3 へのアップロードの両方を確認した。

---

## 目的

* Public IP を持たない Private EC2 を作成する
* SSH を使わずに Session Manager で EC2 を操作する
* SSM 系 Interface Endpoint の役割を理解する
* S3 Gateway Endpoint の役割を理解する
* IAM Policy と VPC Endpoint の違いを理解する
* Private DNS と Interface Endpoint の関係を理解する
* 意図的に設定を壊し、エラーの違いを確認する

---

## 構成図

```text
AWS Cloud
├── Amazon S3
│   └── S3 Bucket
│       ├── test.txt
│       └── ec2_test.txt
│
└── VPC
    └── Private Subnet
        ├── Private EC2
        │   ├── Public IPなし
        │   ├── SSH開放なし
        │   ├── SSM Agent
        │   └── AWS CLI
        │
        ├── SSM用 Interface Endpoint
        │   ├── ssm
        │   ├── ssmmessages
        │   └── ec2messages
        │
        └── Private Subnet用 Route Table
            └── S3向けルート
                ↓
            S3 Gateway Endpoint
```

---

## S3 は Private Subnet 内に置かない

S3 は EC2 や RDS のように、VPC や Subnet 内へ配置するサービスではない。

```text
誤ったイメージ:

VPC
└── Private Subnet
    └── S3
```

実際には、S3 は VPC の外側にある AWS サービス。

```text
正しいイメージ:

VPC
└── Private Subnet
    └── EC2
        ↓
   S3 Gateway Endpoint
        ↓
       S3
```

---

## 作成した主なリソース

### ネットワーク

```text
VPC
Private Subnet
Private Subnet用 Route Table
```

### EC2

```text
Amazon Linux 2023
Public IP: なし
SSH: 開放しない
```

### IAM Role

EC2 用 IAM Role に、以下の AWS 管理ポリシーを付与した。

```text
AmazonSSMManagedInstanceCore
AmazonS3FullAccess
```

役割:

```text
AmazonSSMManagedInstanceCore
→ EC2 内の SSM Agent が Systems Manager と通信するため

AmazonS3FullAccess
→ EC2 内で実行した AWS CLI が S3 を読み書きするため
```

`AmazonS3FullAccess` は権限が広いため、今回は学習用として使用した。
実務では、対象バケットだけに権限を絞る。

### Security Group

```text
Private EC2用 Security Group
Interface Endpoint用 Security Group
```

Endpoint 用 Security Group では、EC2 から届く HTTPS 通信を許可した。

```text
Inbound:
HTTPS / TCP 443
Source: Private EC2用 Security Group
```

### SSM 系 Interface Endpoint

東京リージョンで以下を作成した。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

### S3 Gateway Endpoint

```text
com.amazonaws.ap-northeast-1.s3
```

タイプ:

```text
Gateway
```

Private Subnet 用 Route Table と関連付けた。

---

## 実施内容

### 1. S3 のファイルを EC2 へダウンロード

```bash
aws s3 cp s3://バケット名/test.txt .
```

ファイル内容を確認。

```bash
cat test.txt
```

### 2. EC2 でファイルを作成

```bash
echo "hello from EC2" > ec2_test.txt
```

### 3. EC2 から S3 へアップロード

```bash
aws s3 cp ec2_test.txt s3://バケット名/
```

S3 コンソールでも、`ec2_test.txt` が追加されていることを確認した。

---

## 結果

以下を確認できた。

```text
S3
↓
Private EC2へファイルをダウンロード

Private EC2
↓
S3へファイルをアップロード
```

EC2 には Public IP がなく、NAT Gateway も Internet Gateway 向けルートも存在しない。

それでも、S3 Gateway Endpoint を使うことで、Private EC2 から S3 へアクセスできた。

---

## 症状・原因・解決の要約

| 症状                                               | 原因                                  | 解決                                               |
| ------------------------------------------------ | ----------------------------------- | ------------------------------------------------ |
| `aws` コマンドが使えない                                  | Ubuntu EC2 に AWS CLI が入っていなかった      | AWS CLI が標準で利用できる Amazon Linux 2023 で EC2 を作り直した |
| S3 から EC2 へのコピーで `Permission denied`             | `/usr/bin` に保存しようとしていた              | `cd ~` でホームディレクトリへ移動してからコピーした                    |
| `not authorized to perform: s3:ListAllMyBuckets` | EC2 用 IAM Role から S3 Policy を外した    | S3 用 IAM Policy を再度付与した                          |
| Session Manager でコマンドを入力できない                     | Endpoint 用 SG の HTTPS 443 許可を外した    | Endpoint 用 SG に HTTPS 443 を再度許可した                |
| 再起動後、Session Manager の画面は出るがプロンプトが返らない           | SSM 系 Endpoint の Private DNS を無効化した | Private DNS を再度有効化した                             |

---

## 切り分けで分かったこと

```text
Endpoint
→ AWSサービスまでの通信経路

IAM Policy
→ AWSサービスへ到達した後、何を操作してよいか決める権限

Security Group
→ 通信を通してよいか判断する門

DNS
→ サービス名を Interface Endpoint の Private IP へ変換する案内役
```

権限があっても、経路がなければ通信できない。
経路があっても、権限がなければ S3 の操作は拒否される。

---

## 学び

### Interface Endpoint

```text
Subnet内に AWS サービス用の入口を作る
ENI が作成される
Private IP を持つ
Security Group を付ける
Private DNS が重要
```

今回の例:

```text
SSM
ssmmessages
ec2messages
```

### Gateway Endpoint

```text
Route Table に AWS サービス向けの専用ルートを追加する
ENI は作成されない
Private IP を持たない
Security Group は付けない
```

今回の例:

```text
S3
```

### 一番重要な整理

```text
SSM用 Interface Endpoint
→ Private EC2を Session Manager で操作するため

S3 Gateway Endpoint
→ Private EC2から S3 へアクセスするため
```

---

## 詳細ドキュメント

* [構築手順](docs/setup.md)
* [トラブルシューティングと理解メモ](docs/troubleshooting-and-notes.md)

