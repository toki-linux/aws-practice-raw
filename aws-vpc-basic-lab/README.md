# AWS VPC Basic Lab

## 概要

このリポジトリは、AWSで自作VPCを作成し、その中にパブリックサブネットとEC2インスタンスを配置して、インターネット疎通を確認した学習記録です。

今回は、AWSのデフォルトVPCではなく、自分でVPC・サブネット・Internet Gateway・ルートテーブル・セキュリティグループを作成しました。

目的は、EC2インスタンスがどのようなネットワーク構成の上で動いているのかを理解することです。

---

## 今回のゴール

今回のゴールは以下です。

- 自作VPCを作成する
- パブリックサブネットを作成する
- Internet GatewayをVPCにアタッチする
- ルートテーブルを設定する
- 自作VPC内にEC2インスタンスを起動する
- EC2からインターネットへ通信できることを確認する
- 最後に作成したリソースを削除する

---

## 作成した構成

```text
VPC
└── Public Subnet
    └── EC2 Instance
        ├── Public IPv4 Address
        ├── Security Group
        └── Internet Access
```

より具体的には以下の構成です。

```text
toki-vpc-01
├── CIDR: 10.0.0.0/16
│
├── toki-public-subnet-01
│   └── CIDR: 10.0.1.0/24
│
├── toki-igw-01
│   └── Internet Gateway
│
├── toki-public-rt-01
│   ├── 10.0.0.0/16 → local
│   └── 0.0.0.0/0 → Internet Gateway
│
└── EC2 Instance
    ├── Private IP: 10.0.1.x
    ├── Public IP: あり
    └── Security Group: SSH許可
```

---

## 使用した主なAWSサービス

- VPC
- Subnet
- Internet Gateway
- Route Table
- Security Group
- EC2

---

## VPC作成

VPCを作成しました。

設定内容は以下です。

```text
Name tag: toki-vpc-01
IPv4 CIDR block: 10.0.0.0/16
IPv6 CIDR block: なし
Tenancy: Default
```

VPCは、AWS上に作る自分専用のネットワーク空間です。

EC2インスタンスは単体で存在するのではなく、必ずどこかのVPCの中に配置されます。

---

## サブネット作成

VPC内にパブリックサブネットを作成しました。

設定内容は以下です。

```text
Subnet name: toki-public-subnet-01
VPC: toki-vpc-01
Availability Zone: ap-northeast-1a
IPv4 subnet CIDR block: 10.0.1.0/24
```

VPC全体のCIDRは `10.0.0.0/16` です。

その中の一部として、サブネットに `10.0.1.0/24` を割り当てました。

```text
VPC:      10.0.0.0/16
Subnet:   10.0.1.0/24
```

VPCが大きな箱だとすると、サブネットはその中の小部屋のようなものです。

---

## 自動パブリックIP割り当て

作成したサブネットで、自動パブリックIPv4アドレス割り当てを有効化しました。

```text
Enable auto-assign public IPv4 address: 有効
```

これにより、このサブネット内でEC2を起動したときに、パブリックIPが自動で割り当てられるようになります。

パブリックIPがない場合、外部からSSH接続することができません。

---

## Internet Gateway作成

Internet Gatewayを作成し、VPCにアタッチしました。

```text
Internet Gateway name: toki-igw-01
Attach target: toki-vpc-01
```

Internet Gatewayは、VPCがインターネットと通信するための出口です。

ただし、Internet GatewayをVPCにアタッチしただけでは、サブネットからインターネットへ通信できるわけではありません。

サブネットがInternet Gatewayへ向かうためには、ルートテーブルの設定が必要です。

---

## ルートテーブル作成

パブリックサブネット用のルートテーブルを作成しました。

```text
Route table name: toki-public-rt-01
VPC: toki-vpc-01
```

その後、以下のルートを追加しました。

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
Target detail: toki-igw-01
```

ルートテーブルの状態は以下です。

```text
10.0.0.0/16 → local
0.0.0.0/0  → toki-igw-01
```

`10.0.0.0/16 → local` は、VPC内の通信を表します。

`0.0.0.0/0 → Internet Gateway` は、VPC外のインターネットに向かう通信をInternet Gatewayへ流す設定です。

---

## ルートテーブルとサブネットの関連付け

作成したルートテーブルを、パブリックサブネットに関連付けました。

```text
Route table: toki-public-rt-01
Associated subnet: toki-public-subnet-01
```

これにより、`toki-public-subnet-01` に配置されたEC2は、ルートテーブルの設定に従ってInternet Gateway経由でインターネットへ通信できるようになります。

---

## EC2インスタンス作成

自作VPC内のパブリックサブネットにEC2インスタンスを作成しました。

ネットワーク設定は以下です。

```text
VPC: toki-vpc-01
Subnet: toki-public-subnet-01
Auto-assign public IP: Enable
```

セキュリティグループでは、SSH接続を許可しました。

```text
Inbound rule:
Type: SSH
Protocol: TCP
Port: 22
Source: My IP
```

---

## SSH接続

EC2インスタンス起動後、パブリックIPv4アドレスを確認し、SSH接続しました。

```bash
ssh -i キーペア名.pem ubuntu@パブリックIP
```

接続できたことで、以下が正しく設定されていることを確認できました。

- EC2にパブリックIPが付与されている
- セキュリティグループでSSHが許可されている
- EC2が自作VPC内のサブネットに配置されている

---

## EC2内のIP確認

EC2にSSH接続したあと、以下のコマンドでIPアドレスを確認しました。

```bash
ip addr
```

プライベートIPが `10.0.1.x` になっていることを確認しました。

これは、EC2インスタンスが `10.0.1.0/24` のサブネット内に配置されていることを意味します。

---

## インターネット疎通確認

EC2からインターネットへ通信できるか確認しました。

```bash
curl -I https://aws.amazon.com
```

結果として、HTTPステータスコード `200` が返ってきました。

```text
HTTP/2 200
```

これにより、以下の構成が正しく機能していることを確認できました。

```text
EC2
↓
Public Subnet
↓
Route Table
↓
Internet Gateway
↓
Internet
```

---
## トラブルシューティング：SSH接続できなかった原因

自作VPC内にEC2を作成したあと、SSH接続できない問題が発生しました。

原因は、ルートテーブルに `0.0.0.0/0 → Internet Gateway` は追加していたものの、そのルートテーブルをEC2が所属するサブネットに関連付けていなかったことです。

```text
Routes タブ
0.0.0.0/0 → igw-xxxx
は追加済み

しかし

Subnet associations タブ
対象サブネットにチェックなし
```

つまり、外への道が書かれたルートテーブルは存在していたものの、EC2が所属するサブネットがそのルートテーブルを使っていませんでした。

解決方法は、対象ルートテーブルの `Subnet associations` から、EC2が所属するサブネットを関連付けることでした。

詳しくは以下にまとめています。

- [SSH接続できなかった原因：ルートテーブルのサブネット関連付け漏れ](docs/troubleshooting-ssh-route-table.md)
## 確認できたこと

今回の実践で、以下を確認できました。

- EC2インスタンスはVPCの中に作成される
- サブネットはVPC内のIP範囲を区切ったもの
- パブリックサブネットにするにはInternet Gatewayへのルートが必要
- Internet Gatewayを作成しただけではインターネット通信はできない
- ルートテーブルで `0.0.0.0/0` をInternet Gatewayに向ける必要がある
- EC2にパブリックIPがないと外部からSSH接続できない
- セキュリティグループでSSHを許可しないと接続できない
- 自作VPCでも、正しく設定すればEC2からインターネットへ通信できる

---

## 今回の重要ポイント

パブリックサブネットは、名前で決まるものではありません。

サブネットがパブリックになる条件は、主に以下です。

```text
1. サブネットがVPC内に存在する
2. VPCにInternet Gatewayがアタッチされている
3. サブネットに関連付いたルートテーブルに 0.0.0.0/0 → Internet Gateway がある
4. EC2にパブリックIPがある
5. セキュリティグループで必要な通信が許可されている
```

つまり、サブネット名に `public` と付けただけでは、パブリックサブネットにはなりません。

---

## default セキュリティグループについて

自作VPCを作成したところ、セキュリティグループ一覧に `default` が2つ表示されました。

これは正常な動きです。

セキュリティグループの `default` は、AWSアカウントに1つだけ存在するものではなく、VPCごとに1つ自動作成されます。

```text
Default VPC
└── default Security Group

自作VPC
└── default Security Group
```

そのため、自作VPCを作成すると、そのVPC用の `default` セキュリティグループも自動で作成されます。

見分けるときは、セキュリティグループ名ではなく `VPC ID` を確認します。

---

## 削除について

学習後は、作成したリソースを削除しました。

基本的には、VPC本体を最初に消すのではなく、依存しているリソースから順番に削除します。

削除の考え方は以下です。

```text
中にあるもの
↓
接続しているもの
↓
最後に箱
```

つまり、VPCは最後に削除します。

詳しい削除手順は以下にまとめています。

[cleanup.md](docs/cleanup.md)

---

## 学び

今回の実践で、VPCはEC2によって作られるものではなく、EC2より上位にあるネットワークの土台であることを理解しました。

以前は、EC2を作成するとVPCが一緒に作られるようなイメージがありました。

しかし実際には、以下のような関係です。

```text
VPC
└── Subnet
    └── EC2
```

EC2はVPCの中に置かれるリソースです。

VPCはEC2のためだけにあるものではなく、AWS上でネットワーク範囲を分けるための土台です。

---

## 30秒説明

AWSのVPCは、AWS上に作る自分専用のネットワーク空間です。

今回は自分でVPCを作成し、その中にパブリックサブネットを作り、Internet Gatewayとルートテーブルを設定しました。

そのサブネット内にEC2を起動し、SSH接続したあと、`curl -I https://aws.amazon.com` で外部通信を確認しました。

HTTPステータスコード200が返ってきたため、自作VPC内のEC2からインターネットへ通信できることを確認できました。

この実践を通して、VPC・サブネット・Internet Gateway・ルートテーブル・セキュリティグループの役割を整理できました。
