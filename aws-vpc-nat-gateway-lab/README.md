# AWS VPC NAT Gateway Lab

## 概要

このリポジトリは、AWSで自作VPCを作成し、Private Subnet内のEC2インスタンスからNAT Gateway経由でインターネットへ通信できることを確認した学習記録です。

前回までの学習では、Public SubnetとPrivate Subnetを分け、Public EC2経由でPrivate EC2へSSH接続できる構成を作成しました。

今回はその続きとして、Private EC2から外部サイトへ直接通信しようとすると失敗することを確認し、その後NAT Gatewayを使ってPrivate EC2からインターネットへ出られるようにしました。

---

## 今回のゴール

今回のゴールは以下です。

- Private EC2からインターネットへ直接通信できないことを確認する
- NAT Gatewayを作成する
- Private Route Tableに `0.0.0.0/0 → NAT Gateway` を追加する
- Private EC2から外部サイトへ通信できることを確認する
- NAT Gatewayは外からPrivate EC2へ入る入口ではないことを理解する
- NAT Gateway、Elastic IP、EC2、VPC関連リソースを削除する

---

## 構成

今回の構成は以下です。

```text
VPC: 10.0.0.0/16

├── Public Subnet: 10.0.1.0/24
│   ├── public-ec2
│   │   ├── Public IPあり
│   │   └── 自分のPCからSSH可能
│   │
│   └── Internet Gatewayへのルートあり
│
└── Private Subnet: 10.0.2.0/24
    └── private-ec2
        ├── Public IPなし
        └── NAT Gateway経由で外部通信
```

NAT Gateway追加後の通信経路は以下です。

```text
private-ec2
↓
Private Route Table
↓
NAT Gateway
↓
Internet Gateway
↓
Internet
```

---

## 使用した主なAWSサービス

- VPC
- Public Subnet
- Private Subnet
- Internet Gateway
- Route Table
- Security Group
- EC2
- NAT Gateway
- Elastic IP

---

## 前提構成

NAT Gatewayを作成する前の状態では、以下のようなPublic / Private構成を作成しました。

```text
自分のPC
↓ SSH
public-ec2
↓ SSH
private-ec2
```

### Public Route Table

Public Subnetに関連付けたルートテーブルです。

```text
Destination: 10.0.0.0/16
Target: local

Destination: 0.0.0.0/0
Target: Internet Gateway
```

Public SubnetにはInternet Gatewayへのルートがあるため、Public EC2はインターネットと通信できます。

---

### Private Route Table

NAT Gateway作成前のPrivate Subnet用ルートテーブルです。

```text
Destination: 10.0.0.0/16
Target: local
```

この時点では、Private Subnetには以下のルートがありません。

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

また、NAT Gatewayへのルートもまだありません。

そのため、Private EC2はインターネットへ直接出られません。

---

## 症状

Private EC2にSSH接続したあと、外部サイトへ通信できるか確認しました。

```bash
curl -I https://aws.amazon.com
```

### コマンドの意味

```text
curl
→ 指定したURLへHTTPリクエストを送るコマンド

-I
→ レスポンスヘッダーだけを取得するオプション

https://aws.amazon.com
→ 通信確認に使った外部サイト
```

NAT Gateway作成前は、この通信が失敗しました。

---

## 原因

原因は、Private SubnetのRoute Tableに外部インターネットへ向かうルートがなかったことです。

Private Route Tableは以下の状態でした。

```text
10.0.0.0/16 → local
```

これは、VPC内の通信だけを許可するルートです。

一方で、インターネットへ出るための以下のようなルートはありませんでした。

```text
0.0.0.0/0 → Internet Gateway
```

または、

```text
0.0.0.0/0 → NAT Gateway
```

そのため、Private EC2から外部サイトへ通信しようとしても、通信の行き先がなく失敗しました。

---

## 解決

NAT Gatewayを作成し、Private Route Tableに以下のルートを追加しました。

```text
Destination: 0.0.0.0/0
Target: NAT Gateway
```

これにより、Private EC2からインターネットへ向かう通信はNAT Gatewayへ送られるようになりました。

---

## NAT Gateway作成

AWSコンソールでNAT Gatewayを作成しました。

```text
VPC
→ NAT gateways
→ Create NAT gateway
```

今回の画面では、以下のような設定を使用しました。

```text
アベイラビリティーモード: リージョナル - 新規
接続タイプ: パブリック
Elastic IP割り当て方法: 自動
```

---

## リージョナルNAT Gatewayについて

今回の画面では、従来のNAT Gateway作成手順でよく見る「配置先サブネット」を選ぶ欄がありませんでした。

これは、今回作成したNAT GatewayがリージョナルNAT Gatewayだったためです。

従来のゾーナルNAT Gatewayは、Public Subnetに配置する必要があります。

```text
従来のゾーナルNAT Gateway
→ Public Subnetに配置する
```

一方、今回作成したリージョナルNAT Gatewayでは、サブネットを直接指定せず、VPC単位で作成する形でした。

```text
リージョナルNAT Gateway
→ サブネットを指定しない
→ VPC単位で作成する
→ AWS側が裏側の配置を管理する
```

そのため、作成時にPublic Subnetを選ぶ欄がなくても、今回の構成ではそのまま進められました。

---

## Private Route Tableを編集

NAT Gateway作成後、Private Subnetに関連付いているRoute Tableを編集しました。

```text
VPC
→ Route Tables
→ private-rt
→ Routes
→ Edit routes
→ Add route
```

追加したルートは以下です。

```text
Destination: 0.0.0.0/0
Target: NAT Gateway
```

最終的なPrivate Route Tableは以下です。

```text
Destination: 10.0.0.0/16
Target: local

Destination: 0.0.0.0/0
Target: NAT Gateway
```

---

## 動作確認

Private EC2にSSH接続し、外部通信を確認しました。

```bash
curl -I https://aws.amazon.com
```

結果として、HTTPステータスコード `200` が返りました。

```text
HTTP/2 200
```

これにより、Private EC2からNAT Gateway経由でインターネットへ通信できることを確認できました。

---

## 切り分け

今回の切り分けは以下の流れで行いました。

### 1. Private EC2から外部通信できないことを確認

```bash
curl -I https://aws.amazon.com
```

NAT Gateway作成前は失敗しました。

これにより、Private EC2がそのままではインターネットへ出られないことを確認しました。

---

### 2. Private Route Tableを確認

Private Route Tableには、最初は以下しかありませんでした。

```text
10.0.0.0/16 → local
```

この状態では、VPC内通信はできますが、インターネットへ出るルートがありません。

---

### 3. NAT Gatewayを作成

NAT Gatewayを作成しました。

```text
NAT Gateway
→ Publicな外向き通信のための出口
```

今回の画面ではリージョナルNAT Gatewayだったため、サブネット選択欄はありませんでした。

---

### 4. Private Route TableにNAT Gatewayへのルートを追加

```text
0.0.0.0/0 → NAT Gateway
```

これにより、Private EC2の外向き通信がNAT Gatewayへ向かうようになりました。

---

### 5. 再度curlで確認

```bash
curl -I https://aws.amazon.com
```

`200` が返ったため、NAT Gateway経由で外部通信できていると判断しました。

---

## NAT Gatewayの役割

NAT Gatewayは、Private Subnet内のリソースから外部インターネットへ通信するための出口です。

重要なのは、NAT Gatewayは外からPrivate EC2へ入る入口ではないことです。

```text
private-ec2 → Internet
→ できる

Internet → private-ec2
→ できない
```

NAT Gatewayは、Private EC2から開始した通信を外へ出し、その戻り通信をPrivate EC2へ返す役割を持ちます。

外部からPrivate EC2へ新しく接続を開始するためのものではありません。

---

## Public Subnetとの違い

Public SubnetのEC2は、Public IPを持ち、Route TableにInternet Gatewayへのルートがあれば、インターネットと直接通信できます。

```text
public-ec2
↓
Internet Gateway
↓
Internet
```

一方、Private EC2はPublic IPを持たず、Internet Gatewayへの直接ルートもありません。

その代わり、NAT Gatewayを経由して外へ出ます。

```text
private-ec2
↓
NAT Gateway
↓
Internet Gateway
↓
Internet
```

---

## 削除手順

NAT Gatewayは課金が発生するため、検証後すぐに削除しました。

削除順は以下です。

```text
1. Private Route Tableの 0.0.0.0/0 → NAT Gateway を削除
2. NAT Gatewayを削除
3. Elastic IPが残っていれば解放
4. EC2 2台をTerminate
5. Security Groupを削除
6. Subnetを削除
7. Route Tableを削除
8. Internet GatewayをDetachして削除
9. VPCを削除
```

---

## NAT Gatewayの削除

まず、Private Route TableからNAT Gatewayへのルートを削除しました。

```text
Private Route Table
→ Routes
→ Edit routes
→ 0.0.0.0/0 → NAT Gateway を削除
```

その後、NAT Gatewayを削除しました。

```text
VPC
→ NAT gateways
→ 対象のNAT Gatewayを選択
→ Actions
→ Delete NAT gateway
```

---

## Elastic IPの確認

NAT Gateway削除後、Elastic IPが残っていないか確認しました。

```text
EC2
→ Elastic IPs
```

今回の環境では、NAT Gateway削除後にElastic IPが自動的に消えました。

そのため、追加の解放作業は不要でした。

ただし、環境によってはElastic IPが残る可能性があります。

Elastic IPが残っている場合は、以下から解放します。

```text
EC2
→ Elastic IPs
→ 対象EIPを選択
→ Actions
→ Release Elastic IP addresses
```

---

## 学び

今回の実践で理解したことは以下です。

```text
Private EC2は、そのままだとインターネットへ出られない
理由は、Private Route Tableに外向きのルートがないため
NAT Gatewayを作成し、Private Route Tableに0.0.0.0/0 → NAT Gatewayを追加すると外へ出られる
NAT Gatewayは外向き通信のための出口
NAT Gatewayは外からPrivate EC2へ入る入口ではない
今回のリージョナルNAT Gatewayでは、サブネット選択欄がなかった
NAT Gatewayは課金があるため、検証後すぐ削除する
Elastic IPが残っていないか確認する
```

---

## 30秒説明

Private Subnet内のEC2からインターネットへ通信できるか検証しました。

最初はPrivate Route Tableに `10.0.0.0/16 → local` しかなかったため、Private EC2から `curl -I https://aws.amazon.com` を実行しても外部通信できませんでした。

その後、NAT Gatewayを作成し、Private Route Tableに `0.0.0.0/0 → NAT Gateway` を追加しました。

再度Private EC2からcurlを実行したところ、HTTPステータスコード200が返り、NAT Gateway経由でインターネットへ出られることを確認できました。

この実践を通して、NAT GatewayはPrivate EC2から外へ出るための出口であり、外部からPrivate EC2へ直接入る入口ではないことを理解しました。

---

## 詳細メモ

より詳しい手順や切り分けは以下にまとめています。

- [NAT GatewayでPrivate EC2から外部通信する](docs/nat-gateway-private-subnet.md)
