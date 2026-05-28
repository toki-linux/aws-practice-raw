# README.md

# NAT GatewayなしでPrivate EC2へSSM Session Manager接続する

## 概要

このリポジトリは、AWS Systems Manager の Session Manager を使って、Private Subnet 内の EC2 インスタンスへ SSH なしで接続した実践記録です。

今回の構成では、以下を使いません。

- Public IP
- SSH
- 踏み台サーバ
- Internet Gateway
- NAT Gateway

代わりに、以下を使います。

- AWS Systems Manager
- Session Manager
- SSM Agent
- IAM Role
- Interface VPC Endpoint
- Security Group

## 目的

Private Subnet 内の EC2 に対して、インターネット経由ではなく VPC Endpoint 経由で AWS Systems Manager に接続し、Session Manager から操作できることを確認する。

## 構成イメージ

```text
VPC: 10.0.0.0/16
└── Private Subnet: 10.0.1.0/24
    ├── Private EC2
    │   ├── Public IPなし
    │   ├── SSHポート開放なし
    │   ├── IAM Roleあり
    │   └── SSM Agentあり
    │
    ├── VPC Endpoint: ssm
    ├── VPC Endpoint: ssmmessages
    └── VPC Endpoint: ec2messages
```

通信イメージ。

```text
Private EC2 内の SSM Agent
↓ HTTPS 443
VPC Endpoint
↓
AWS Systems Manager
↓
Session Manager
↓
ブラウザからEC2を操作
```

## 今回作成したリソース

### IAM Role

```text
Role name:
ec2-ssm-role

Policy:
AmazonSSMManagedInstanceCore
```

EC2 内の SSM Agent が AWS Systems Manager と通信するための権限を渡す。

### VPC

```text
Name:
ssm-private-vpc

IPv4 CIDR:
10.0.0.0/16
```

### Subnet

```text
Name:
private-subnet-1a

Availability Zone:
ap-northeast-1a

IPv4 CIDR:
10.0.1.0/24
```

### Security Group

```text
private-ec2-sg
→ Private EC2用

ssm-endpoint-sg
→ VPC Endpoint用
```

### VPC Endpoint

以下3つの Interface VPC Endpoint を作成した。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

### EC2

```text
Subnet:
private-subnet-1a

Public IP:
なし

Security Group:
private-ec2-sg

IAM Role:
ec2-ssm-role
```

## 手順

## 1. IAM Roleを作成

EC2用のIAM Roleを作成した。

```text
Trusted entity type:
AWS service

Use case:
EC2

Policy:
AmazonSSMManagedInstanceCore

Role name:
ec2-ssm-role
```

このRoleにより、EC2内のSSM AgentがAWS Systems Managerと通信できるようになる。

ポイントは、IAM RoleがSSM Agentを起動するためのものではないこと。

IAM Roleは、SSM AgentがAWS Systems Managerとやり取りするための許可証として使われる。

```text
SSM Agent
= EC2内の連絡係

IAM Role
= 連絡係がAWS Systems Managerと通信するための許可証
```

## 2. VPCを作成

```text
Name:
ssm-private-vpc

IPv4 CIDR:
10.0.0.0/16
```

今回はInternet Gatewayを作成しない。

理由は、Private EC2をインターネットへ直接出さず、VPC Endpoint経由でAWS Systems Managerに接続する構成にするため。

## 3. Private Subnetを作成

```text
Name:
private-subnet-1a

Availability Zone:
ap-northeast-1a

IPv4 CIDR:
10.0.1.0/24
```

今回作成するEC2は、このPrivate Subnetに配置する。

## 4. Security Groupを作成

### private-ec2-sg

Private EC2用のSecurity Group。

```text
Inbound:
なし

Outbound:
All traffic
```

SSH接続を使わないため、Inboundで22番ポートは開けない。

### ssm-endpoint-sg

VPC Endpoint用のSecurity Group。

```text
Inbound:
HTTPS / TCP 443 / Source: private-ec2-sg

Outbound:
All traffic
```

Private EC2からVPC EndpointへHTTPS通信できるようにする。

```text
Private EC2
↓ HTTPS / TCP 443
VPC Endpoint
```

## 5. VPCのDNS設定を有効化

VPC EndpointでPrivate DNSを有効にするため、VPC側のDNS設定を有効化した。

```text
DNS resolution:
有効

DNS hostnames:
有効
```

ここでいうDNSは、独自ドメインを持っているかどうかとは関係ない。

VPC内でAWSサービス名を名前解決するための設定。

例。

```text
ssm.ap-northeast-1.amazonaws.com
```

この名前を、インターネット側ではなくVPC Endpoint側へ向けるために必要。

## 6. VPC Endpointを作成

以下3つのInterface VPC Endpointを作成した。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

共通設定。

```text
VPC:
ssm-private-vpc

Subnet:
private-subnet-1a

Security Group:
ssm-endpoint-sg

Private DNS:
有効

Policy:
Full access
```

それぞれの役割。

```text
ssm
→ Systems Manager本体への入口

ssmmessages
→ Session Managerの接続セッション用

ec2messages
→ EC2上のSSM Agentがメッセージを受け取る用
```

## 7. Private EC2を作成

EC2をPrivate Subnetに作成した。

主な設定。

```text
AMI:
Ubuntu Server

Subnet:
private-subnet-1a

Auto-assign public IP:
Disabled

Security Group:
private-ec2-sg

IAM Role:
ec2-ssm-role
```

今回はSSH接続を使わないため、Security GroupでSSH / TCP 22は許可しない。

## 8. Session Managerで接続

EC2インスタンスを選択し、以下から接続した。

```text
EC2
↓
対象インスタンスを選択
↓
Connect
↓
Session Manager
↓
Connect
```

接続に成功した。

## 接続後の確認コマンド

Session Managerのターミナルで以下を実行した。

### whoami

```bash
whoami
```

説明。

現在のログインユーザーを確認する。

実行結果。

```text
ssm-user
```

Session Manager経由で接続すると、通常は `ssm-user` としてログインする。

### hostname

```bash
hostname
```

説明。

接続しているEC2インスタンスのホスト名を確認する。

Session ManagerでPrivate EC2に入れていることを確認するために使った。

### ip a

```bash
ip a
```

説明。

EC2のネットワークインターフェースとIPアドレスを確認する。

Public IPではなく、Private IPのみで動作していることを確認するために使った。

### curlで外部通信を確認

```bash
curl -I https://aws.amazon.com
```

説明。

EC2からインターネット上のWebサイトへHTTPリクエストできるか確認する。

今回はNAT GatewayもInternet Gatewayも作成していないため、通常のインターネット通信はできない。

この確認により、以下の状態を確認できた。

```text
インターネット全体には出られない
でもVPC Endpoint経由でSystems Managerには接続できる
```

## 症状

通常、Private Subnet内のEC2はPublic IPを持たないため、自分のPCから直接SSH接続できない。

また、今回は以下も作成していない。

- Public EC2
- 踏み台サーバ
- Internet Gateway
- NAT Gateway

そのため、普通に考えるとPrivate EC2へ接続できない状態になる。

```text
自分のPC
↓ SSH
Private EC2

→ Public IPなし
→ SSHポート開放なし
→ 接続できない
```

## 原因

Private EC2へSSH接続するには、通常以下が必要になる。

```text
Public IP
SSHポート22の許可
秘密鍵
sshd
通信経路
```

しかし今回の構成では、それらを使わない。

代わりに、EC2内のSSM AgentがAWS Systems Managerと通信し、そのセッションをAWSコンソールから利用する。

```text
Private EC2内のSSM Agent
↓
AWS Systems Manager
↓
Session Manager
↓
ブラウザから操作
```

そのため、SSHとは別の仕組みでEC2へ接続できる。

## 解決

以下の3つをそろえることで、NAT Gatewayなし・SSHなしでPrivate EC2に接続できた。

```text
1. EC2内にSSM Agentがある
2. EC2にSSM用のIAM Roleが付いている
3. EC2からSystems Managerへ行く通信経路がある
```

今回、3つ目の通信経路をVPC Endpointで作成した。

```text
Private EC2
↓
VPC Endpoint
↓
AWS Systems Manager
```

## 切り分け

Session Managerに接続候補が出ない場合は、以下を確認する。

### 1. EC2が起動しているか

```text
Instance state:
Running

Status checks:
2/2 checks passed
```

起動直後は反映に時間がかかることがあるため、数分待つ。

### 2. EC2にIAM Roleが付いているか

```text
IAM Role:
ec2-ssm-role
```

EC2にIAM Roleが付いていないと、SSM AgentがAWS Systems Managerと通信する権限を持てない。

### 3. IAM Roleに必要なポリシーがあるか

```text
Policy:
AmazonSSMManagedInstanceCore
```

このポリシーがないと、EC2がSystems Managerの管理対象として動けない。

### 4. VPC Endpointが3つあるか

```text
ssm
ssmmessages
ec2messages
```

それぞれ `Available` になっているか確認する。

### 5. EndpointのSecurity Groupで443を許可しているか

```text
Inbound:
HTTPS / TCP 443 / Source: private-ec2-sg
```

Private EC2からVPC EndpointへHTTPS通信できる必要がある。

### 6. Private DNSが有効か

VPC Endpoint作成時に、Private DNSを有効化する。

```text
Private DNS:
有効
```

### 7. VPC側のDNS設定が有効か

```text
DNS resolution:
有効

DNS hostnames:
有効
```

Private DNSを使うためには、VPC側のDNS設定も必要。

### 8. SSM Agentが入っているAMIか

UbuntuのAWS公式AMIではSSM Agentが入っていることが多い。

ただし、入っているだけでは足りない。

```text
SSM Agentあり
IAM Roleあり
VPC Endpointあり
```

この3つがそろう必要がある。

## 削除手順

検証後、以下の順で削除した。

```text
1. Session Managerの接続を終了
2. EC2インスタンスをTerminate
3. VPC Endpoint 3つを削除
   - ssm
   - ssmmessages
   - ec2messages
4. Security Groupを削除
   - private-ec2-sg
   - ssm-endpoint-sg
5. Subnetを削除
6. VPCを削除
7. IAM Role ec2-ssm-role を削除
```

補足。

VPCを作成すると、Main Route Table や Network ACL などが自動作成される。

自分でRoute Tableを作成していない場合、VPC削除時に一緒に削除される。

## 学び

今回の実践で、Private EC2に対してPublic IPやSSHを使わずに接続できることを確認した。

特に重要だったのは以下。

```text
SSM Agent
= EC2内で動くSystems Managerとの連絡係

IAM Role
= SSM AgentがAWS Systems Managerと通信するための許可証

VPC Endpoint
= Private SubnetからAWSサービスへ行くための専用窓口

Security Group
= 通信を通してよいか判断する門番

Session Manager
= SSHなしでEC2へ接続する機能
```

SSH接続では、自分のPCからEC2へ直接入る。

```text
自分のPC
↓ SSH / TCP 22
EC2
```

Session Managerでは、自分のPCからEC2へ直接SSHしない。

```text
自分のブラウザ
↓
AWS Systems Manager
↓
SSM Agent
↓
EC2
```

そのため、以下が不要になる。

```text
Public IP
SSHポート開放
秘密鍵
踏み台サーバ
```

## 一文まとめ

Private EC2にPublic IPやSSHを用意しなくても、EC2内のSSM AgentがVPC Endpoint経由でAWS Systems Managerと通信できれば、Session Managerを使ってAWSコンソールからPrivate EC2に接続できる。
