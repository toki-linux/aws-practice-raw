# トラブルシューティングと理解メモ

## 1. 全体像

今回作成した構成:

```text
Private EC2
├── SSM系 Interface Endpoint
│   └── Session ManagerでEC2を操作するため
│
└── S3 Gateway Endpoint
    └── S3へアクセスするため
```

EC2 はインターネットへ直接接続しない。

```text
Public IP: なし
NAT Gateway: なし
Internet Gateway向けルート: なし
```

そのため、AWS サービスとの通信には VPC Endpoint を使用した。

---

# 基本概念

## 2. IAM Role と IAM Policy

IAM Role は、EC2 に必要な権限を持たせるために使用する。

```text
EC2
└── IAM Role
    ├── AmazonSSMManagedInstanceCore
    └── AmazonS3FullAccess
```

### AmazonSSMManagedInstanceCore

```text
EC2内のSSM Agent
↓
IAM Roleの権限を利用
↓
Systems Managerと通信
```

### AmazonS3FullAccess

```text
EC2内で実行したAWS CLI
↓
IAM Roleの権限を利用
↓
S3を操作
```

IAM Role と Endpoint は別物。

```text
IAM Role
→ 操作してよいか決める許可証

Endpoint
→ AWSサービスへ到達するための道
```

---

## 3. Security Group

Security Group は、通信を通すか判断する門。

今回作成したもの:

```text
Private EC2用SG
Interface Endpoint用SG
```

### Private EC2用SG

```text
Inbound:
なし

Outbound:
すべて許可
```

外部から EC2 へ直接接続しないため、SSH や HTTP を開ける必要はない。

### Interface Endpoint用SG

```text
Inbound:
HTTPS / TCP 443
Source: Private EC2用SG
```

実際の通信:

```text
Private EC2
↓ HTTPS / TCP 443
Interface EndpointのENI
```

Security Group 自体が通信を送信するわけではない。

```text
EC2
→ 通信する主体

EndpointのENI
→ 通信を受け取る入口

Endpoint用SG
→ 通信を許可する門
```

---

## 4. Security Group はステートフル

EC2 がステートレスなのではない。

```text
Security Groupがステートフル
```

EC2 から外向き通信を開始した場合、戻りの通信は自動的に許可される。

```text
EC2
↓ リクエスト送信
AWSサービス
↓ 応答
EC2へ戻れる
```

戻りの通信を Inbound Rule に個別で書く必要はない。

---

## 5. Interface Endpoint

Interface Endpoint は、Subnet 内に AWS サービス向けの入口を作る。

```text
Private Subnet
├── EC2
│
└── Interface Endpoint
    └── ENI
        └── Private IP
```

特徴:

```text
Subnetを指定する
ENIが作成される
Private IPを持つ
Security Groupを付ける
Private DNSが重要
```

今回使用したサービス:

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

---

## 6. Gateway Endpoint

Gateway Endpoint は、Route Table に AWS サービス向けの専用ルートを追加する。

```text
Private EC2
↓
Private Subnet用 Route Table
↓
S3向け Prefix List
↓
S3 Gateway Endpoint
↓
S3
```

特徴:

```text
Subnetを直接指定しない
ENIを作らない
Private IPを持たない
Security Groupを付けない
Route Tableと関連付ける
```

今回使用したサービス:

```text
com.amazonaws.ap-northeast-1.s3
```

タイプ:

```text
Gateway
```

---

## 7. Interface Endpoint と Gateway Endpoint の違い

| 項目                 | Interface Endpoint | Gateway Endpoint |
| ------------------ | ------------------ | ---------------- |
| 今回の用途              | SSM 系サービス          | S3               |
| Subnet 指定          | する                 | しない              |
| Route Table との関連付け | 基本不要               | 必要               |
| ENI                | 作成される              | 作成されない           |
| Private IP         | 持つ                 | 持たない             |
| Security Group     | 付ける                | 付けない             |
| DNS                | Private DNS が重要    | Route Table が重要  |
| イメージ               | Private な受付窓口      | 専用道路             |

覚え方:

```text
Interface Endpoint
→ Subnet内にPrivate IP付きの受付窓口を作る

Gateway Endpoint
→ Route TableにAWSサービス専用道路を追加する
```

---

## 8. DNS と Private DNS

SSM Agent は、IP アドレスを直接指定せず、AWS サービス名を使って通信する。

例:

```text
ssm.ap-northeast-1.amazonaws.com
ssmmessages.ap-northeast-1.amazonaws.com
```

DNS は、サービス名を IP アドレスへ変換する。

```text
AWSサービス名
↓
DNSで名前解決
↓
Interface EndpointのPrivate IP
```

Interface Endpoint の Private DNS を有効にすると、通常の AWS サービス名を使ったまま、Interface Endpoint 経由で通信できる。

### なぜ最初から IP アドレスを使わないのか

```text
IPアドレス
→ 将来変更される可能性がある

ドメイン名
→ DNS側で新しいIPへ案内できる
```

サービス名と IP アドレスを切り離すことで、接続先の変更へ柔軟に対応できる。

---

## 9. Session Manager の画面と EC2 内のシェル

ブラウザに表示される黒いターミナル画面と、EC2 内で実際に動くシェルは別物。

```text
ブラウザ
└── AWSコンソール上のターミナル画面
        ↓ 入力
   Session Manager
        ↓
   SSM Agent
        ↓
   EC2内のシェル
        ↓ 実行結果
   SSM Agent
        ↓
ブラウザに表示
```

ターミナル画面はブラウザ側で表示できる。

ただし、EC2 内のシェルとの通信が成立しなければ、プロンプトは返ってこない。

```text
ターミナル画面が表示される
≠
EC2への接続成功
```

プロンプトが返り、コマンドを実行できて初めて正常接続と判断できる。

---

# 症状・原因・解決

## 10. Ubuntu で `aws` コマンドが使えなかった

### 症状

```bash
aws --version
```

実行結果:

```text
command not found
```

### 原因

Ubuntu EC2 に AWS CLI がインストールされていなかった。

### 解決

EC2 を Amazon Linux 2023 で作り直した。

Amazon Linux 2023 では AWS CLI v2 が標準で利用できる。

### 学び

```text
IAM RoleやEndpointが存在しても、
AWS CLI自体がなければ aws s3 コマンドは使えない
```

---

## 11. S3 から EC2 へのコピーで `Permission denied`

### 症状

```bash
aws s3 cp s3://バケット名/test.txt .
```

実行時に `Permission denied` が表示された。

### 原因

コマンド実行時の現在地が `/usr/bin` だった。

最後の `.` は、現在いるディレクトリを意味する。

```text
S3のtest.txt
↓
/usr/binへ保存しようとした
↓
一般ユーザーには書き込み権限がない
↓
Permission denied
```

### 切り分け

現在地を確認する。

```bash
pwd
```

説明:

```text
現在いるディレクトリを確認する
```

### 解決

ホームディレクトリへ移動した。

```bash
cd ~
```

再度コピー。

```bash
aws s3 cp s3://バケット名/test.txt .
```

### 学び

```text
Permission denied
→ 必ずしもIAM Policyの問題ではない

Linux側のファイル権限や保存先も確認する
```

---

## 12. S3 Gateway Endpoint を削除した

### 実験

S3 Gateway Endpoint を削除した。

```text
com.amazonaws.ap-northeast-1.s3
```

### 症状

S3 へのアクセスができなくなった。

### 原因

Private Subnet 用 Route Table から、S3 向けルートが消えた。

```text
Private EC2
↓
S3へ行く道がない
↓
アクセス失敗
```

### 解決

S3 Gateway Endpoint を再作成し、Private Subnet 用 Route Table と関連付けた。

### 学び

```text
IAM Policyが残っていても、
通信経路がなければS3へ到達できない
```

---

## 13. IAM Role から S3 Policy を外した

### 実験

EC2 用 IAM Role から S3 用 Policy を外した。

### 症状

```bash
aws s3 ls
```

実行結果:

```text
not authorized to perform: s3:ListAllMyBuckets
because no identity-based policy allows
the s3:ListAllMyBuckets action
```

### 原因

S3 Gateway Endpoint は残っていたため、S3 までの通信経路は存在した。

しかし、EC2 が利用する IAM Role に S3 操作権限がなかった。

```text
道はある
↓
S3へ到達できる
↓
許可証がない
↓
操作を拒否される
```

### 解決

EC2 用 IAM Role に S3 用 Policy を再度付与した。

### 学び

```text
Endpoint
→ 通信経路

IAM Policy
→ 操作権限
```

この2つは別物。

---

## 14. Endpoint 用 SG から HTTPS 443 を外した

### 実験

Interface Endpoint 用 Security Group から、以下の Inbound Rule を削除した。

```text
HTTPS / TCP 443
Source: Private EC2用SG
```

### 症状

Session Manager で接続中のターミナルから、コマンドを入力できなくなった。

### 原因

EC2 から Interface Endpoint の ENI へ向かう HTTPS 通信が、Security Group で遮断された。

```text
EC2
×
Interface Endpoint
```

### 解決

Endpoint 用 Security Group に、HTTPS 443 の Inbound Rule を再度追加した。

### 学び

```text
Interface Endpointは、
接続開始時だけでなく、
接続後の通信経路としても使われる
```

---

## 15. SSM 系 Endpoint の Private DNS を無効化した

### 実験

以下の Interface Endpoint で Private DNS を無効化した。

```text
ssm
ssmmessages
ec2messages
```

### 最初の症状

すでに接続中だった Session Manager では、すぐには操作不能にならなかった。

### 原因

既存の通信チャネルが残っていた可能性がある。

```text
DNSでEndpointのIPを調べる
↓
接続を確立する
↓
既存接続を使い続ける
```

接続確立後、コマンドを入力するたびに DNS 問い合わせをやり直すとは限らない。

### 再起動後の症状

EC2 を再起動して新しく Session Manager へ接続すると、黒いターミナル画面は表示されたが、プロンプトは返ってこなかった。

### 原因

Private DNS が無効なため、SSM Agent が Interface Endpoint の Private IP を正常に見つけられなかった可能性が高い。

```text
SSM Agent
↓
AWSサービス名を使う
↓
Private DNSが無効
↓
Interface EndpointのPrivate IPへ案内されない
↓
通信チャネルを正常に作れない
```

### 解決

SSM 系 Interface Endpoint の Private DNS を再度有効にした。

その後、設定反映を待ち、EC2 を再起動して新規接続を確認した。

### 学び

```text
DNS
→ 接続先を見つける案内役

Interface Endpoint
→ 実際に通信を流す経路
```

Private DNS を壊した場合、既存接続ではすぐに影響が出ないことがある。
新規接続や再起動後の再接続で確認する。

---

# 切り分けの整理

## 16. 症状ごとに疑う場所

| 症状                                | 疑う場所                                 |
| --------------------------------- | ------------------------------------ |
| `aws: command not found`          | AWS CLI が入っているか                      |
| `Permission denied`               | Linux 側の保存先・ファイル権限                   |
| `not authorized to perform ...`   | IAM Role・IAM Policy                  |
| S3 への通信がタイムアウトする                  | S3 Gateway Endpoint・Route Table      |
| Session Manager で操作不能             | Endpoint 用 SG の HTTPS 443            |
| 再起動後に Session Manager のプロンプトが返らない | Private DNS・SSM 系 Interface Endpoint |
| Interface Endpoint を作ったのに接続できない   | ENI・SG・Private DNS・VPC の DNS 設定      |

---

## 17. 役割の違い

```text
VPC
→ AWS上に仮想ネットワーク空間を作る

Subnet
→ VPCのIPアドレス範囲を区切る

Route Table
→ 通信の宛先ごとに、次にどこへ送るか決める

IAM Role
→ EC2に権限を渡す

IAM Policy
→ 許可する操作内容を定義する

Security Group
→ 通信を通してよいか判断する

DNS
→ サービス名をIPアドレスへ変換する

Interface Endpoint
→ Subnet内にPrivate IP付きのAWSサービス向け入口を作る

Gateway Endpoint
→ Route TableにS3などへの専用ルートを追加する

SSM Agent
→ Systems Managerからの要求をEC2内で処理する

AWS CLI
→ EC2内からAWSサービスを操作するコマンドツール
```

---

# 最後の学び

## 18. 一番重要な理解

```text
通信できるためには、
道・門・権限がそろう必要がある
```

今回の場合:

```text
道
→ VPC Endpoint

門
→ Security Group

権限
→ IAM Role / IAM Policy
```

さらに Interface Endpoint では、入口まで案内する DNS も重要。

```text
DNS
→ Interface EndpointのPrivate IPを案内する
```

---

## 19. 今回説明できるようになったこと

```text
Private EC2へ入るために、
SSM系 Interface Endpointを使う。

Interface EndpointはSubnet内にENIを作り、
Private IPを持つ入口になる。

Private EC2からS3へ行くために、
S3 Gateway Endpointを使う。

Gateway EndpointはENIを作らず、
Route TableにS3向けルートを追加する。

Endpointは通信経路。
IAM Policyは操作権限。
Security Groupは通信を許可する門。
DNSはInterface EndpointのPrivate IPへ案内する。
```

