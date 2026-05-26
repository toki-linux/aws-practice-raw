# AWS VPC Public / Private Subnet Lab

## 概要

このリポジトリは、AWSで自作VPCを作成し、Public SubnetとPrivate Subnetを分けてEC2を配置した学習記録です。

Public Subnetに配置したEC2には自分のPCからSSH接続し、Private Subnetに配置したEC2にはPublic EC2経由でSSH接続しました。

今回の目的は、Public SubnetとPrivate Subnetの違いを実際に構成して理解することです。

---

## 今回のゴール

今回のゴールは以下です。

- 自作VPCを作成する
- Public Subnetを作成する
- Private Subnetを作成する
- Public用とPrivate用でルートテーブルを分ける
- Public EC2には自分のPCからSSH接続できるようにする
- Private EC2には外部から直接SSHできない構成にする
- Public EC2からPrivate EC2へPrivate IPでSSH接続する
- 最後に秘密鍵とAWSリソースを削除する

---

## 作成した構成

```text
VPC: 10.0.0.0/16

├── Public Subnet: 10.0.1.0/24
│   └── public-ec2
│       ├── Public IPあり
│       ├── Private IP: 10.0.1.x
│       └── 自分のPCからSSH可能
│
└── Private Subnet: 10.0.2.0/24
    └── private-ec2
        ├── Public IPなし
        ├── Private IP: 10.0.2.x
        └── public-ec2からのみSSH可能
```

通信の流れは以下です。

```text
自分のPC
↓ SSH
public-ec2
↓ SSH（Private IP宛て）
private-ec2
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

自作VPCを作成しました。

```text
Name: toki-vpc-public-private-01
CIDR: 10.0.0.0/16
```

VPCはAWS上に作る自分専用のネットワーク空間です。

EC2はVPCの中に直接置かれるのではなく、VPC内のサブネットに配置されます。

```text
VPC
└── Subnet
    └── EC2
```

---

## Public Subnet作成

外部からアクセスするEC2を配置するため、Public Subnetを作成しました。

```text
Name: toki-public-subnet-01
CIDR: 10.0.1.0/24
Auto-assign public IPv4: ON
```

Public Subnetに配置するEC2には、外部からSSH接続するためにPublic IPを付与します。

---

## Private Subnet作成

外部から直接アクセスしないEC2を配置するため、Private Subnetを作成しました。

```text
Name: toki-private-subnet-01
CIDR: 10.0.2.0/24
Auto-assign public IPv4: OFF
```

Private EC2にはPublic IPを付けません。

Private Subnetは、外部から直接アクセスさせたくないリソースを配置する場所として使います。

---

## Internet Gateway作成

VPCにInternet Gatewayを作成し、アタッチしました。

```text
Name: toki-igw-01
Attach: toki-vpc-public-private-01
```

Internet Gatewayは、VPCがインターネットと通信するための出口です。

ただし、Internet GatewayをVPCにアタッチしただけでは、すべてのサブネットがインターネットに出られるわけではありません。

サブネットがInternet Gatewayを使うには、ルートテーブルで設定する必要があります。

---

## ルートテーブル作成

今回は、Public Subnet用とPrivate Subnet用でルートテーブルを分けました。

理由は、Public SubnetとPrivate Subnetで通信ルールが異なるためです。

---

## Public用ルートテーブル

Public Subnet用のルートテーブルを作成しました。

```text
Name: toki-public-rt-01
```

ルートは以下です。

```text
Destination: 10.0.0.0/16
Target: local

Destination: 0.0.0.0/0
Target: Internet Gateway
```

`0.0.0.0/0 → Internet Gateway` があるため、このルートテーブルを使うサブネットはインターネットへ通信できます。

このルートテーブルをPublic Subnetに関連付けました。

```text
Subnet associations:
toki-public-subnet-01
```

---

## Private用ルートテーブル

Private Subnet用のルートテーブルを作成しました。

```text
Name: toki-private-rt-01
```

ルートは以下です。

```text
Destination: 10.0.0.0/16
Target: local
```

Private用ルートテーブルには、以下のルートを入れませんでした。

```text
0.0.0.0/0 → Internet Gateway
```

これにより、Private Subnetはインターネットへ直接出られない構成になります。

このルートテーブルをPrivate Subnetに関連付けました。

```text
Subnet associations:
toki-private-subnet-01
```

---

## ルートテーブルの考え方

最初は、ルートテーブルは地図のようなものなので、1枚にできるだけ情報を書いた方がよいと思っていました。

しかし、今回の実践を通して、ルートテーブルは1枚にまとめるものではなく、通信ルールごとに分けるものだと理解しました。

```text
同じ通信ルールのサブネット
→ 同じルートテーブルを使う

通信ルールが違うサブネット
→ 別のルートテーブルを使う
```

今回の場合は、Public SubnetとPrivate Subnetで通信ルールが違います。

```text
Public Subnet
→ インターネットへ出したい
→ 0.0.0.0/0 → Internet Gateway が必要

Private Subnet
→ インターネットへ直接出したくない
→ 0.0.0.0/0 → Internet Gateway は不要
```

そのため、ルートテーブルを以下の2つに分けました。

```text
public-rt
private-rt
```

---

## EC2作成

EC2は2台作成しました。

```text
public-ec2
private-ec2
```

---

## public-ec2

public-ec2は、自分のPCからSSH接続する入口として作成しました。

```text
Name: public-ec2
Subnet: toki-public-subnet-01
Public IP: あり
Private IP: 10.0.1.x
AMI: Ubuntu
```

Security Groupは以下です。

```text
public-ec2-sg

Inbound:
SSH / TCP 22 / My IP
```

これにより、自分のPCからpublic-ec2へSSH接続できます。

---

## private-ec2

private-ec2は、外部から直接アクセスさせないEC2として作成しました。

```text
Name: private-ec2
Subnet: toki-private-subnet-01
Public IP: なし
Private IP: 10.0.2.x
AMI: Ubuntu
```

Security Groupは以下です。

```text
private-ec2-sg

Inbound:
SSH / TCP 22 / Source: public-ec2-sg
```

これにより、public-ec2-sgが付いているEC2からのみ、private-ec2へSSH接続できるようになります。

---

## Security Groupの考え方

Security Groupは、そのEC2に対して「誰が入ってきてよいか」を決める入口ルールです。

今回の構成では、通信の流れが以下になります。

```text
自分のPC
↓
public-ec2
↓
private-ec2
```

そのため、許可する通信元もEC2ごとに異なります。

```text
public-ec2-sg
→ 自分のPCのIPからSSHを許可

private-ec2-sg
→ public-ec2-sgからSSHを許可
```

private-ec2側に `My IP` を許可するのではなく、public-ec2のSecurity GroupをSourceに指定する点が重要です。

---

## Security Group指定で詰まった点

private-ec2-sgのInbound ruleで、Sourceにpublic-ec2-sgを指定しようとした際、以下のようなエラーが出ました。

```text
既存の IPv4 CIDR ルールに 1 つの 参照先のグループ ID を指定することはできません。
```

原因は、既存のIPv4 CIDRルールに対して、Security Group IDを指定しようとしていたことです。

Sourceには以下のように種類があります。

```text
IPv4 CIDR
例: 203.x.x.x/32
例: 10.0.1.0/24

Security Group
例: sg-xxxxxxxxxxxxxxxxx
```

既存のIPv4 CIDRルールを編集してSecurity Group IDを入れようとするのではなく、SSHルールを新しく追加することで解決しました。

最終的には以下のように設定しました。

```text
private-ec2-sg

Inbound:
Type: SSH
Protocol: TCP
Port: 22
Source: public-ec2-sg
```

---

## SSH接続確認

まず、自分のPCからpublic-ec2へSSH接続しました。

```bash
ssh -i key.pem ubuntu@public-ec2のPublicIP
```

これは成功しました。

---

## public-ec2からprivate-ec2へSSH接続

次に、public-ec2からprivate-ec2へSSH接続しました。

private-ec2にはPublic IPがないため、Private IP宛てに接続します。

```bash
ssh -i private-key.pem ubuntu@private-ec2のPrivateIP
```

今回、public-ec2とprivate-ec2では同じキーペアを使用していました。

ただし、public-ec2からprivate-ec2へSSHするには、public-ec2上でprivate-ec2用の秘密鍵を使える状態にする必要があります。

そのため、自分のPCからpublic-ec2へ秘密鍵を一時的にコピーしました。

```bash
scp -i key.pem key.pem ubuntu@public-ec2のPublicIP:/home/ubuntu/private-key.pem
```

public-ec2上で秘密鍵の権限を変更しました。

```bash
chmod 400 /home/ubuntu/private-key.pem
```

その後、private-ec2へSSH接続しました。

```bash
ssh -i /home/ubuntu/private-key.pem ubuntu@private-ec2のPrivateIP
```

これにより、public-ec2経由でprivate-ec2へSSH接続できました。

---

## known_hosts と authorized_keys の違い

SSH接続時に、最初はknown_hostsにprivate-ec2の公開鍵を置く必要があるのではないかと考えました。

しかし、SSH認証で重要なのはknown_hostsではなく、接続先サーバ側のauthorized_keysです。

```text
known_hosts
→ 接続先サーバが本物か確認するための記録

authorized_keys
→ この秘密鍵を持っている人ならログインしてよい、という認証用の公開鍵置き場
```

public-ec2からprivate-ec2へ初めてSSHすると、接続先のホスト確認メッセージが出ることがあります。

```text
The authenticity of host '10.0.2.x' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

ここで `yes` と入力すると、private-ec2のホスト鍵情報がpublic-ec2のknown_hostsに保存されます。

これは認証用ではなく、次回以降も同じ接続先か確認するためのものです。

---

## Permission denied の原因

public-ec2からprivate-ec2へSSH接続しようとした際、最初は以下のエラーが出ました。

```text
Permission denied
```

原因は、private-ec2用の秘密鍵をpublic-ec2上に置けていなかったことです。

public-ec2からprivate-ec2へSSHする場合、public-ec2上でprivate-ec2用の秘密鍵を使える必要があります。

```text
自分のPC
↓ key.pemでSSH
public-ec2
↓ private-key.pemでSSH
private-ec2
```

今回は同じキーペアを使用していたため、自分のPCにある秘密鍵をpublic-ec2へコピーして解決しました。

---

## 秘密鍵の扱い

学習用として、public-ec2に秘密鍵を一時的にコピーしてSSH接続を確認しました。

ただし、秘密鍵はEC2へログインするための重要な情報です。

そのため、接続確認後はpublic-ec2上から秘密鍵を削除しました。

```bash
rm /home/ubuntu/private-key.pem
```

削除確認も行いました。

```bash
ls -l /home/ubuntu/private-key.pem
```

以下のように表示されれば削除できています。

```text
No such file or directory
```

今回のように踏み台EC2へ秘密鍵を置く方法は、学習用途では理解しやすいですが、実務では慎重に扱う必要があります。

実務寄りの方法としては、以下のような方法もあります。

```text
SSH Agent Forwarding
SSM Session Manager
```

---

## Ubuntuのログインメッセージ

private-ec2へSSH接続した際、Ubuntuのアップデートに関するメッセージが表示されました。

これはUbuntuのログイン時に表示される通常の案内です。

```text
Welcome to Ubuntu
System information
updates can be applied
```

このメッセージが表示されたことで、private-ec2へのSSHログイン自体は成功していると判断できます。

ただし、今回のprivate-ec2はNAT GatewayなしのPrivate Subnetに配置しているため、基本的にはインターネットへ出られません。

そのため、`sudo apt update` などは失敗する可能性があります。

---

## pingが通らなかったこと

private-ec2からpublic-ec2のPrivate IPへpingを実行しましたが、通りませんでした。

ただし、今回の目的はpublic-ec2経由でprivate-ec2へSSH接続することだったため、今回はここで深追いしませんでした。

pingはSSHとは異なり、ICMPを使用します。

```text
SSH  = TCP 22
ping = ICMP
```

そのため、pingを通したい場合は、public-ec2側のSecurity GroupでICMPを許可する必要があります。

例:

```text
public-ec2-sg

Inbound:
All ICMP - IPv4 / Source: private-ec2-sg
```

今回はSSH接続の確認を目的としていたため、ICMP設定は次回以降の課題としました。

---

## 削除手順

学習後、作成したリソースを削除しました。

削除前に、まずprivate-ec2から抜けました。

```bash
exit
```

public-ec2に戻ったあと、public-ec2上の秘密鍵を削除しました。

```bash
rm /home/ubuntu/private-key.pem
```

その後、public-ec2からも抜けました。

```bash
exit
```

AWSコンソールからEC2 2台を終了しました。

```text
public-ec2
private-ec2
```

その後、ネットワーク関連リソースを順番に削除しました。

```text
1. EC2 2台をTerminate
2. 自作Security Groupを削除
3. Public Subnet / Private Subnetを削除
4. Public Route Table / Private Route Tableを削除
5. Internet GatewayをDetachして削除
6. VPCを削除
```

VPCはネットワークの土台なので、最後に削除しました。

---

## 今回確認できたこと

今回の実践で、以下を確認できました。

- Public SubnetとPrivate Subnetを分けて作成できる
- Public SubnetにはInternet Gatewayへのルートを持たせる
- Private SubnetにはInternet Gatewayへのルートを持たせない
- ルートテーブルは通信ルールごとに分ける
- Public EC2にはPublic IPを付ける
- Private EC2にはPublic IPを付けない
- Private EC2には自分のPCから直接SSHしない
- Public EC2からPrivate EC2へPrivate IPでSSHできる
- Security GroupのSourceに別のSecurity Groupを指定できる
- SSHでは通信許可と鍵認証の両方が必要
- known_hostsとauthorized_keysは役割が違う
- 踏み台EC2に秘密鍵を置く場合は確認後に削除する

---

## 学び

今回の実践で、Private Subnetは「完全に孤立した場所」ではなく、同じVPC内のリソースとは通信できる場所だと理解しました。

Private SubnetのEC2にはPublic IPがなく、Internet Gatewayへのルートもありません。

そのため、自分のPCから直接SSHすることはできません。

しかし、同じVPC内にあるpublic-ec2からであれば、Private IPを使ってprivate-ec2へSSH接続できます。

```text
自分のPC
↓
public-ec2
↓
private-ec2
```

この構成により、外部から直接アクセスさせたくないEC2をPrivate Subnetに配置し、必要なときだけPublic EC2経由で接続するイメージを掴めました。

また、ルートテーブルは1枚にすべての情報を書くものではなく、通信ルールごとに分け、サブネットに関連付けるものだと理解しました。

---

 30秒説明

AWSで自作VPCを作成し、Public SubnetとPrivate Subnetを分けてEC2を配置しました。

Public SubnetのEC2にはPublic IPを付け、自分のPCからSSH接続できるようにしました。

Private SubnetのEC2にはPublic IPを付けず、Internet Gatewayへのルートも設定しませんでした。

その代わり、private-ec2のSecurity Groupでpublic-ec2-sgからのSSHだけを許可し、public-ec2からprivate-ec2のPrivate IPへSSH接続しました。

この実践を通して、Public SubnetとPrivate Subnetの違い、ルートテーブルの分け方、Security Groupによる通信元制御、踏み台経由でPrivate EC2へ接続する流れを理解しました。
