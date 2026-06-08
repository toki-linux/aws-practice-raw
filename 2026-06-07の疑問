
# VPC Endpoint・Route Table・Session Managerの通信経路

## 概要

このファイルでは、以下の内容を整理する。

```text
Public Subnetの条件
Internet Gateway向けルートを削除した場合
TCP通信の考え方

Interface Endpoint
Gateway Endpoint
Route Tableとの関係

Private EC2
SSM Agent
Session Manager
VPC Endpoint
```

---

# Public Subnetとは？

## 概要

Public Subnetは、Internet Gatewayへ向かうルートを持つSubnet。

```text
Public Subnet
=
Internet Gatewayへ向かうルートを持つSubnet
```

代表的なRoute Tableは以下。

| Destination | Target |
|---|---|
| `10.0.0.0/16` | `local` |
| `0.0.0.0/0` | `igw-xxxxxxxx` |

---

## 各ルートの意味

```text
10.0.0.0/16 → local
→ VPC内の通信に使う

0.0.0.0/0 → Internet Gateway
→ VPC外のインターネットへ向かう通信に使う
```

---

## EC2がインターネット通信する条件

Public SubnetへEC2を置くだけでは、インターネット通信できるとは限らない。

主に以下が必要。

```text
Internet GatewayをVPCへアタッチする

Subnetが使うRoute Tableへ
0.0.0.0/0 → Internet Gateway
を設定する

EC2へPublic IPv4アドレスまたはElastic IPを付ける

Security GroupやNetwork ACLで必要な通信を許可する
```

---

# Internet Gateway向けルートを削除するとどうなるか？

## 問題

Public Subnet用のRoute Tableから、以下のルートを削除する。

```text
0.0.0.0/0 → Internet Gateway
```

この状態で、ブラウザからEC2のWebページへアクセスするとどうなるか。

---

## 結論

ブラウザからEC2のWebページへ正常にアクセスできない。

一般的には、ブラウザでタイムアウトする。

```text
0.0.0.0/0 → Internet Gateway
を削除
↓
Subnetからインターネットへの経路がなくなる
↓
EC2とブラウザの間で正常な通信が成立しない
↓
Webページを表示できない
```

---

## 「ブラウザからEC2へ届くが、戻りだけ失敗する」と考えてよいか？

完全にそう断定しない方がよい。

確実に言えるのは、以下。

```text
EC2とインターネットの間で、
行きと帰りを含めた正常な通信が成立しない
```

少なくとも、EC2からインターネット上のブラウザへ返事を送るためのルートがなくなる。

---

# TCP通信として考える

ブラウザからWebページへアクセスする場合、最初からHTMLファイルを受け取るわけではない。

先にTCP接続を成立させる必要がある。

簡略化すると以下。

```text
1. ブラウザ
   「接続したいです」
   ↓

2. EC2
   「接続できます」
   ↓

3. ブラウザ
   「了解です」
   ↓

4. HTTPリクエストを送る
   ↓

5. EC2がWebページを返す
```

---

## ルートを削除した場合

```text
EC2
↓
ブラウザへ返事を送りたい
↓
0.0.0.0/0 → Internet Gateway がない
↓
VPC外へ返事を出せない
```

そのため、仮に最初の通信がEC2付近まで到達しても、TCP接続が正常に完成しない。

---

## 満点回答

```text
Public Subnet用のRoute Tableから
0.0.0.0/0 → Internet Gateway
を削除すると、EC2とインターネットの間で正常な通信が成立しなくなる。

Webページへアクセスするには、
ブラウザとEC2の間で行きと帰りの通信が必要。

少なくともEC2からブラウザへ応答を返すための経路がなくなるため、
TCP接続を正常に確立できず、Webページは表示されない。

ブラウザからEC2へ最初の通信が届いたとしても、
戻りの通信を含めた一連の通信は成立しない。
```

---

# VPC Endpointとは？

## 概要

VPC Endpointは、VPC内のリソースからAWSサービスへ、インターネットを経由せずに接続するための仕組み。

```text
VPC Endpoint
=
VPC内からAWSサービスへ
プライベートに接続するための入口・経路
```

---

## なぜ使うのか？

Private Subnet内のEC2には、Public IPやNAT Gatewayを用意しない場合がある。

```text
Private EC2
├── Public IPなし
├── NAT Gatewayなし
└── Internet Gatewayへ向かう経路なし
```

この状態では、インターネット経由でAWSサービスへ接続できない。

VPC Endpointを作ると、AWSネットワーク内でAWSサービスへ接続できる。

---

# VPC Endpointの代表的な種類

初心者が最初に押さえるのは、以下の2種類。

```text
Interface Endpoint
Gateway Endpoint
```

---

# Interface Endpointとは？

## 概要

Interface Endpointは、Subnet内にPrivate IPを持つENIを作り、そのENIを入口としてAWSサービスへ接続する方式。

```text
Interface Endpoint
=
Subnet内へPrivate IP付きのENIを作る方式
```

---

## 構成

```text
Private Subnet
└── Interface Endpoint
    └── ENI
        ├── Private IP
        └── Security Group
```

---

## 通信経路

```text
Private EC2
↓ HTTPS : 443
Interface EndpointのENI
↓
AWSサービス
```

EC2は、Subnet内やVPC内にあるENIのPrivate IPへ通信する。

---

## Route Tableとの関係

Interface Endpointでは、Gateway Endpointのような専用ルートをRoute Tableへ追加しない。

```text
Interface Endpoint
→ ENIのPrivate IPへ通信する
→ 専用ルートをRoute Tableへ追加しない
```

ただし、Route Tableと無関係という意味ではない。

VPC内通信なので、通常は既存の `local` ルートが使われる。

```text
10.0.0.0/16 → local
```

---

## DNSとの関係

Private DNSを有効にすると、通常のAWSサービス名が、Interface EndpointのENIが持つPrivate IPへ名前解決される。

```text
ssm.ap-northeast-1.amazonaws.com
↓ DNSで名前解決
Interface EndpointのENIのPrivate IP
```

EC2は、名前解決されたPrivate IPへ通信する。

---

## Interface Endpointで使うもの

```text
ENI
Private IP
Security Group
DNS
```

---

# Gateway Endpointとは？

## 概要

Gateway Endpointは、Route Tableへ専用ルートを追加し、S3やDynamoDBへ接続する方式。

```text
Gateway Endpoint
=
Route Tableへ専用ルートを追加する方式
```

---

## 対応する代表的なサービス

```text
Amazon S3
Amazon DynamoDB
```

---

## 構成

```text
Private Subnet
↓
Route Table
├── 10.0.0.0/16 → local
└── pl-xxxxxxxx → vpce-xxxxxxxx
↓
S3
```

---

## Prefix Listとは？

`pl-xxxxxxxx` は、Prefix List。

AWSサービスが使用するIPアドレス範囲をまとめたもの。

```text
Prefix List
=
AWSサービスが使うIPアドレス範囲をまとめたリスト
```

たとえば、S3用のPrefix Listがある。

```text
pl-xxxxxxxx
→ 対象リージョンのS3が利用するIPアドレス範囲
```

---

## Route Tableとの関係

Gateway Endpointを作成すると、選択したRoute Tableへ以下のようなルートが追加される。

| Destination | Target |
|---|---|
| `pl-xxxxxxxx` | `vpce-xxxxxxxx` |

S3へ向かう通信は、この専用ルートを通る。

---

# Interface EndpointとGateway Endpointの比較

| 項目 | Interface Endpoint | Gateway Endpoint |
|---|---|---|
| 接続方式 | Private IPを持つENIを作る | Route Tableへ専用ルートを追加する |
| ENI | 作る | 作らない |
| Private IP | 持つ | 持たない |
| Security Group | 付ける | 付けない |
| Route Tableへ専用ルート | 追加しない | 追加する |
| 主な接続先 | SSMなど多数のAWSサービス | S3・DynamoDB |
| 主な通信先 | ENIのPrivate IP | Prefix Listに該当するAWSサービス |

---

## 覚え方

```text
Interface Endpoint
→ Private IP付きの受付窓口を作る

Gateway Endpoint
→ Route Tableへ専用道路を追加する
```

---

# Session Managerとは？

## 概要

Session Managerは、AWS Systems Managerの機能の一つ。

SSH用ポートを外部へ開けずに、EC2へ接続できる。

```text
Session Manager
=
SSHポートを開けずに
EC2へ接続・操作するための仕組み
```

---

## メリット

```text
EC2のInboundでSSH用ポート22を開けなくてよい
踏み台サーバを用意しなくてよい
SSH鍵を管理しなくてよい
IAMで接続権限を管理できる
```

---

# SSM Agentとは？

## 概要

SSM Agentは、EC2内で動くソフトウェア。

AWS Systems Managerからの依頼を受け取り、EC2内で処理を行う。

```text
SSM Agent
=
EC2内で動くSystems Managerの連絡係
```

---

## 重要なポイント

SSM Agent側から、AWS側へ通信を開始する。

```text
SSM Agent
↓ HTTPS : 443
AWS Systems Manager
```

外部からEC2へ直接SSH接続する方式とは異なる。

そのため、EC2側でInboundのTCP 22を許可する必要がない。

---

# Private EC2とSession Managerの通信経路

## 基本構成

```text
自分のブラウザ
↓
Session Manager
↓
AWS側でセッションを仲介
↕
Interface Endpoint
↕ HTTPS : 443
SSM Agent
↓
Private EC2内でコマンドを実行
```

---

## EC2側から見る

```text
Private EC2
└── SSM Agent
    ↓ HTTPS : 443
    Interface EndpointのENI
    ↓
    AWS Systems Manager
    ↓
    Session Manager
    ↓
    自分のブラウザ
```

---

## 自分側から見る

```text
自分のブラウザ
↓
Session Managerで接続を開始
↓
AWS側でセッションを仲介
↕
SSM Agentが確立した通信経路
↕
Private EC2
```

---

## なぜ双方向に操作できるのか？

SSM AgentがAWS側へHTTPS通信を開始し、セッション用の安全な通信経路を確立する。

その確立済みの通信経路を利用して、コマンドや実行結果を双方向でやり取りする。

```text
SSM Agent
↓
AWS側へ接続を開始
↓
安全なセッション用通信経路を確立
↓
確立済みの経路で双方向通信
```

---

# Session Managerで利用するEndpoint

Private EC2から、NAT Gatewayやインターネットを使わずにSession Managerを利用する場合、Systems Manager向けのInterface Endpointを作る。

基本として押さえるものは以下。

```text
com.amazonaws.<region>.ssm
com.amazonaws.<region>.ssmmessages
```

古い構成との互換性も考慮する場合、以下も登場する。

```text
com.amazonaws.<region>.ec2messages
```

---

## 役割

| Endpoint | 主な役割 |
|---|---|
| `ssm` | Systems ManagerのAPI操作などに使う |
| `ssmmessages` | SSM AgentとSession Manager間の安全な通信チャネルなどに使う |
| `ec2messages` | 以前から使われてきたメッセージ配信用Endpoint |

---

## ec2messagesの注意点

新しいSSM Agentでは、利用可能な場合に `ssmmessages` を優先して使う。

また、2024年以降に開設されたリージョンでは、`ec2messages` はサポートされない。

学習時は以下のように整理する。

```text
ssm
→ Systems Managerとの基本的な連携

ssmmessages
→ Session Managerの通信で重要

ec2messages
→ 古い構成との互換性で登場することがある
```

東京リージョンなどで幅広い互換性を意識した構築例では、3種類を作成する場合がある。

---

# S3 Gateway Endpointが登場する理由

SSM Agentの更新や、S3上のファイル取得などでS3への接続が必要になる場合がある。

```text
Private EC2
↓
S3 Gateway Endpoint
↓
S3
```

ただし、Session Managerの対話接続そのものでは、主に `ssm` と `ssmmessages` のInterface Endpointが重要。

```text
Session Managerの通信
→ Interface Endpoint

S3上のファイル取得・更新など
→ S3 Gateway Endpoint
```

---

# Security Groupの考え方

## Private EC2側

SSM AgentからAWS側へ通信を開始するため、通常はSSH用のInboundを開けない。

```text
Private EC2用Security Group

Inbound
→ SSH用TCP 22は不要

Outbound
→ HTTPS : 443を許可
```

---

## Interface Endpoint側

Interface EndpointのENIへ、EC2からHTTPSで接続できるようにする。

```text
Endpoint用Security Group

Inbound
→ TCP : 443
→ Source：Private EC2用Security Group
```

---

# 全体のまとめ

```text
Public Subnet
→ Internet Gateway向けルートを持つSubnet

0.0.0.0/0 → Internet Gateway
→ インターネットへ向かう通信経路

このルートを削除
→ EC2とブラウザの間で正常な通信が成立しない
→ Webページは表示できない

Interface Endpoint
→ Subnet内へPrivate IP付きENIを作る
→ Security Groupを付ける
→ 専用ルートは追加しない
→ 通常はVPCのlocalルートでENIへ到達する

Gateway Endpoint
→ Route Tableへ専用ルートを追加する
→ S3・DynamoDB向け
→ Prefix Listを使う

Session Manager
→ SSHポートを開けずにEC2へ接続する仕組み

SSM Agent
→ EC2内で動くSystems Managerの連絡係
→ AWS側へHTTPS通信を開始する

ssm
→ Systems Managerの基本的な連携

ssmmessages
→ Session Managerの通信で重要

ec2messages
→ 古い構成との互換性で登場することがある

S3 Gateway Endpoint
→ SSM Agent更新やS3上のファイル取得などで使う場合がある
```


