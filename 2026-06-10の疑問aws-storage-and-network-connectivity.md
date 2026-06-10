
# AWSストレージ・VPC接続・ネットワークセキュリティの整理

## 概要

このファイルでは、以下の内容を整理する。

```text
Amazon FSx for Windows File Server
SMB
Active Directory

VPC Peering
AWS Transit Gateway

AWS Network Firewall

AWS PrivateLink
VPC Endpoint
Interface Endpoint
Gateway Endpoint
```

---

# Amazon FSx for Windows File Serverとは？

## 概要

Amazon FSx for Windows File Serverは、Windows環境向けの共有ファイルストレージサービス。

```text
Amazon FSx for Windows File Server
=
Windows用の共有フォルダサービス
```

Microsoft Windows Server上に構築されたファイルストレージを、AWSがマネージドサービスとして提供する。

---

## 基本構成

```text
Windows PC
↓
ネットワーク
↓
FSx for Windows File Server
↓
共有フォルダへアクセス
```

---

## どのような場面で使うのか？

会社内で、複数の利用者が同じファイルを共有する場面を考える。

```text
営業部
↓
共有フォルダ
↓
見積書を保存

経理部
↓
共有フォルダ
↓
請求書を確認
```

FSx for Windows File Serverを使うと、AWS上にWindows向けの共有フォルダを作れる。

---

# EBS・EFS・FSx for Windowsの違い

| サービス | 主な用途 | 最初の覚え方 |
|---|---|---|
| Amazon EBS | EC2へ接続するブロックストレージ | EC2用ディスク |
| Amazon EFS | Linux系環境で使いやすい共有ファイルストレージ | Linux向け共有フォルダ |
| FSx for Windows File Server | Windows環境向け共有ファイルストレージ | Windows向け共有フォルダ |

---

## イメージ

```text
EBS
→ 1台のEC2で使うディスクに近い

EFS
→ Linux系の複数EC2から共有するファイル置き場

FSx for Windows
→ Windows PCやWindows Serverから使う共有フォルダ
```

---

# なぜWindows向けに別のサービスがあるのか？

Windows環境では、以下のような仕組みがよく使われる。

```text
SMB
Active Directory
Windowsのアクセス権
```

FSx for Windows File Serverは、これらのWindows環境で一般的な機能に対応している。

---

# SMB共有とは？

## 概要

SMBは、`Server Message Block` の略。

Windows環境で共有フォルダへアクセスするときによく使われる通信ルール。

```text
SMB
=
Windowsで共有フォルダへアクセスするためのプロトコル
```

---

## 例

会社PCで以下のようなパスを使って共有フォルダへアクセスすることがある。

```text
\\fileserver\share
```

このようなWindowsの共有フォルダで使われる仕組みがSMB。

---

# Active Directoryとは？

## 概要

Active Directoryは、Windows環境のユーザーやPC、権限などをまとめて管理する仕組み。

略して、`AD` と呼ばれる。

```text
Active Directory
=
会社内のWindowsユーザー・PC・権限を管理する仕組み
```

---

## 何を管理するのか？

```text
ユーザー名
パスワード
所属グループ
PC
アクセス権限
ログイン可否
```

---

## 会社で例える

```text
Active Directory
→ 社員名簿 + 入館管理システム
```

誰が社員なのかを確認し、その社員がどの部屋や資料へアクセスできるかを管理する。

---

# Active Directory連携とは？

## 概要

FSx for Windows File Serverは、Active Directoryと連携できる。

```text
Active Directory連携
=
Windowsのユーザー・権限管理と
FSxの共有フォルダをつなぐこと
```

---

## 例

```text
Active Directory
├── 営業部グループ
├── 経理部グループ
└── 管理者グループ
```

共有フォルダ側で、以下のような制御を行える。

```text
営業部フォルダ
→ 営業部のみアクセス可能

経理部フォルダ
→ 経理部のみアクセス可能
```

---

# VPC Peeringとは？

## 概要

VPC Peeringは、2つのVPCを直接接続する仕組み。

```text
VPC Peering
=
2つのVPCを直接つなぐ接続
```

---

## 構成

```text
VPC-A
↓
VPC Peering
↓
VPC-B
```

接続すると、Private IPを使ってVPC同士で通信できる。

---

## 注意点

1本のPeering接続は、2つのVPCをつなぐ。

```text
VPC-A
↔
VPC-B
```

複数のPeering接続を作ることは可能。

```text
VPC-A ↔ VPC-B
VPC-A ↔ VPC-C
VPC-A ↔ VPC-D
```

ただし、VPC Peeringは推移的ルーティングに対応しない。

---

## 推移的ルーティングとは？

以下の構成を考える。

```text
VPC-A
↔ Peering
VPC-B
↔ Peering
VPC-C
```

この場合、VPC-AからVPC-Cへ、VPC-Bを中継して自動的に通信できるわけではない。

```text
VPC-A
↓
VPC-Bを経由
↓
VPC-C

これは不可
```

VPC-AとVPC-Cを直接つなぎたい場合は、別のPeering接続が必要。

```text
VPC-A
↔ Peering
VPC-C
```

---

# AWS Transit Gatewayとは？

## 概要

AWS Transit Gatewayは、複数のVPCやオンプレミス環境を集中的につなぐ中継ハブ。

```text
AWS Transit Gateway
=
複数VPCや拠点をつなぐ集線装置
```

---

## 構成

```text
             VPC-A
               │
               │
VPC-B ─── Transit Gateway ─── VPC-C
               │
               │
        オンプレミス環境
```

---

## なぜ必要なのか？

VPCが増えると、Peering接続を1本ずつ作る方式では複雑になる。

```text
VPC-A ↔ VPC-B
VPC-A ↔ VPC-C
VPC-A ↔ VPC-D
VPC-B ↔ VPC-C
VPC-B ↔ VPC-D
VPC-C ↔ VPC-D
```

Transit Gatewayを使うと、各VPCを中央のハブへつなげる。

```text
VPC-A
↓
Transit Gateway

VPC-B
↓
Transit Gateway

VPC-C
↓
Transit Gateway
```

---

# VPC PeeringとTransit Gatewayの違い

| 項目 | VPC Peering | Transit Gateway |
|---|---|---|
| 基本構成 | 2つのVPCを直接つなぐ | 中央ハブへ複数VPCを接続する |
| 向いている規模 | 少数のVPC | 多数のVPCや拠点 |
| 推移的ルーティング | 不可 | 可能 |
| 管理方法 | 接続ごとに個別管理 | 中央でまとめて管理しやすい |

---

## 覚え方

```text
VPC Peering
→ VPC同士を直接つなぐ専用通路

Transit Gateway
→ 複数VPCをつなぐ中央駅
```

---

# AWS Network Firewallとは？

## 概要

AWS Network Firewallは、VPC内の通信を検査・制御するマネージド型のネットワークファイアウォール。

```text
AWS Network Firewall
=
VPC内に置く本格的な検問所
```

---

## 主な役割

```text
通信を検査する
不正な通信を遮断する
許可した通信だけを通す
侵入検知・侵入防止を行う
ログを記録する
```

---

# Network Firewallはどこに置くのか？

## 重要なポイント

Network Firewallを作っただけでは、すべての通信が自動的に通るわけではない。

検査したい通信がFirewall Endpointを通るように、Route Tableを設定する。

```text
通信
↓
Route Table
↓
Firewall Endpoint
↓
AWS Network Firewall
↓
接続先
```

---

## インターネット向け通信を検査する例

```text
Private Subnet
↓
Route Table
↓
Network Firewall Endpoint
↓
NAT Gateway
↓
Internet Gateway
↓
インターネット
```

---

## 外部から入る通信を検査する例

```text
インターネット
↓
Internet Gateway
↓
Route Table
↓
Network Firewall Endpoint
↓
EC2など
```

---

# Security Group・NACL・Network Firewallの違い

| 種類 | 主な適用範囲 | 特徴 |
|---|---|---|
| Security Group | ENI、EC2、RDSなど | リソース単位の通信制御 |
| Network ACL | Subnet | Subnet単位の簡易的な通信制御 |
| Network Firewall | VPC内の指定した通信経路 | 詳細な検査ができる本格的なファイアウォール |

---

## 検問所で例える

```text
Network Firewall
→ 詳細な荷物検査を行う本格的な検問所

Network ACL
→ Subnetの入口で確認する簡易的な検問所

Security Group
→ EC2やRDSなど、個別リソースの入口にある門番
```

---

## 必ずこの順番になるのか？

以下のような順番になる構成は作れる。

```text
Network Firewall
↓
Network ACL
↓
Security Group
↓
EC2
```

ただし、常に自動でこの順番になるわけではない。

Network Firewallを通すには、Route Tableで経路を設定する必要がある。

```text
Network Firewall
→ Route Tableで通過させる通信を決める

Network ACL
→ Subnet単位で適用される

Security Group
→ EC2やENIなどのリソース単位で適用される
```

---

# AWS PrivateLinkとは？

## 概要

AWS PrivateLinkは、インターネットを経由せずに、VPCからサービスへプライベートに接続するための仕組み。

```text
AWS PrivateLink
=
プライベート接続を実現する仕組み
```

---

## 接続先の例

```text
AWSサービス
他のVPCで公開されているサービス
他社が提供するサービス
```

---

## イメージ

```text
Private EC2
↓
PrivateLinkを利用した接続
↓
AWSサービスや他社サービス
```

---

# VPC Endpointとは？

## 概要

VPC Endpointは、VPC内に作るAWSサービスへの接続口。

```text
VPC Endpoint
=
VPC内に作るプライベート接続用の入口
```

---

# PrivateLinkとVPC Endpointの違い

```text
PrivateLink
→ プライベート接続を実現する仕組み・技術

VPC Endpoint
→ VPC内に作る具体的な接続口
```

道路で例えると、以下。

```text
PrivateLink
→ 専用道路を利用できる仕組み

VPC Endpoint
→ 専用道路へ入る入口
```

---

# Interface Endpointとは？

## 概要

Interface Endpointは、PrivateLinkを利用するVPC Endpoint。

Subnet内にPrivate IP付きのENIを作る。

```text
Interface Endpoint
=
PrivateLinkを使う接続口
=
Subnet内にENIとPrivate IPを作る
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
Interface EndpointのPrivate IP
↓
AWSサービス
```

---

# Gateway Endpointとは？

## 概要

Gateway Endpointは、S3やDynamoDBへ接続するためのVPC Endpoint。

PrivateLinkとは別の方式。

```text
Gateway Endpoint
=
Route Tableへ専用ルートを追加する接続方式
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
Private EC2
↓
Route Table
↓
Gateway Endpoint
↓
S3・DynamoDB
```

---

# Interface EndpointとGateway Endpointの違い

| 項目 | Interface Endpoint | Gateway Endpoint |
|---|---|---|
| PrivateLinkを使うか | 使う | 使わない |
| ENI | 作る | 作らない |
| Private IP | 持つ | 持たない |
| Security Group | 付ける | 付けない |
| Route Tableへ専用ルート | 追加しない | 追加する |
| 主な用途 | SSMなど多数のAWSサービス | S3・DynamoDB |

---

# 全体のまとめ

```text
FSx for Windows File Server
→ Windows向けの共有フォルダサービス

SMB
→ Windowsで共有フォルダへアクセスするときに使う通信ルール

Active Directory
→ Windowsユーザー・PC・権限を管理する仕組み

VPC Peering
→ 2つのVPCを直接つなぐ
→ 推移的ルーティングはできない

Transit Gateway
→ 複数VPCや拠点をつなぐ中央ハブ

Network Firewall
→ VPC内の通信を詳しく検査する本格的な検問所
→ Route Tableで通信経路へ組み込む

PrivateLink
→ プライベート接続を実現する仕組み

VPC Endpoint
→ VPC内へ作る接続口

Interface Endpoint
→ PrivateLinkを使う
→ ENIとPrivate IPを作る

Gateway Endpoint
→ S3・DynamoDB向け
→ Route Tableへ専用ルートを追加する
```


