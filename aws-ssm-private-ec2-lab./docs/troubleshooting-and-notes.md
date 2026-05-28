

# SSM Session Manager と VPC Endpoint の理解メモ

## 概要

このドキュメントは、NAT GatewayなしでPrivate EC2へSSM Session Manager接続した際の理解メモです。

主に以下を整理する。

- SSMとは何か
- Session Managerとは何か
- SSM Agentの役割
- IAM Roleが必要な理由
- VPC Endpointのイメージ
- Security GroupとIAMの違い
- DNS関連で詰まったポイント

## SSMとは

SSMは Systems Manager の略。

正式名称は以下。

```text
AWS Systems Manager
```

ただし、SSMという略称は昔の名前の名残。

現在のAWS Systems Managerの中に、Session Managerという機能がある。

```text
AWS Systems Manager
└── Session Manager
    └── EC2へSSHなしで接続する機能
```

## Session Managerとは

Session Managerは、AWS Systems Managerの中にある接続機能。

通常のSSHとは違い、EC2に直接22番ポートで入るのではなく、AWS Systems Manager経由でEC2を操作する。

```text
自分のブラウザ
↓
AWS Systems Manager
↓
SSM Agent
↓
EC2
```

## SSM Agentとは

SSM Agentは、EC2の中で動く連絡係。

```text
SSM Agent
= EC2内で動くAWS Systems Managerとの連絡係
```

SSM AgentがAWS Systems Managerと通信することで、Session Manager接続が成立する。

ただし、SSM Agentが入っているだけでは足りない。

必要なものは以下。

```text
1. SSM Agent
2. IAM Role
3. Systems Managerへ届く通信経路
```

## IAM Roleの役割

IAM Roleは、SSM Agentを起動するためのものではない。

正確には、SSM AgentがAWS Systems Managerと通信してよい権限をEC2に渡すもの。

```text
SSM Agent
= 連絡係

IAM Role
= 連絡係がAWS Systems Managerとやり取りするための許可証
```

今回使った管理ポリシー。

```text
AmazonSSMManagedInstanceCore
```

このポリシーを付けたIAM RoleをEC2にアタッチすることで、EC2上のSSM AgentがSystems Managerとやり取りできるようになる。

## 人間側の権限とEC2側の権限

Session Manager接続には、2種類の権限が関係する。

```text
人間側の権限
→ 自分のIAMユーザーがSession Managerを使ってよい

EC2側の権限
→ EC2上のSSM AgentがSystems Managerと通信してよい
```

つまり、EC2側のIAM Roleだけで全てが成立するわけではない。

AWSでは、人間もEC2などのサービスも、AWSサービスを利用するには権限が必要。

```text
人間がAWSを操作する
→ 人間側のIAM権限が必要

EC2がAWSサービスを使う
→ EC2に付けたIAM Roleの権限が必要
```

## VPC Endpointとは

VPC Endpointは、VPC内からAWSサービスへプライベートに接続するための入口。

```text
VPC Endpoint
= AWSサービスへ行く専用の受付窓口
```

今回のイメージ。

```text
Private EC2
↓
VPC内のSSM用受付窓口
↓
AWS Systems Manager
```

NAT Gatewayのようにインターネットへ出るのではなく、指定したAWSサービスへプライベートに接続する。

## NAT GatewayとVPC Endpointの違い

### NAT Gateway

```text
Private EC2
↓
NAT Gateway
↓
Internet Gateway
↓
Internet / AWSサービス
```

Private EC2からインターネット全体へ出るための出口。

### VPC Endpoint

```text
Private EC2
↓
VPC Endpoint
↓
特定のAWSサービス
```

Private EC2から特定のAWSサービスへ接続するための専用窓口。

## 今回必要だったVPC Endpoint

Session ManagerでPrivate EC2に接続するために、以下3つを作成した。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

それぞれの役割。

```text
ssm
→ Systems Manager本体への入口

ssmmessages
→ Session Managerのセッション通信用

ec2messages
→ SSM Agentがメッセージを受け取るため
```

## Private DNSとは

VPC Endpoint作成時に、Private DNSを有効にした。

これは独自ドメインを持っているかどうかとは関係ない。

Private DNSを有効にすると、EC2が通常のAWSサービス名へアクセスしようとしたときに、VPC Endpoint側へ向かうように名前解決される。

例。

```text
ssm.ap-northeast-1.amazonaws.com
```

この名前が、インターネット側ではなく、VPC EndpointのPrivate IPへ解決されるイメージ。

## VPCのDNS設定

Private DNSを有効にするには、VPC側で以下が有効になっている必要がある。

```text
DNS resolution
DNS hostnames
```

今回、VPC Endpoint作成時にDNS関連のエラーが出たため、VPC側のDNS設定を有効化した。

## 症状

VPC Endpoint作成時に、Private DNS関連のエラーが出た。

自分は独自ドメインを持っていないため、最初は「DNSを持っていないから作れないのか」と考えた。

## 原因

原因は独自ドメインの有無ではなかった。

VPC側のDNS設定が有効になっていなかったため、Private DNSを有効にしたVPC Endpointを作成できなかった。

## 解決

VPCの設定から、以下を有効化した。

```text
DNS resolution:
有効

DNS hostnames:
有効
```

その後、VPC Endpoint作成時にPrivate DNSを有効化して作成できた。

## 切り分け

DNS関連で詰まった場合は、以下を確認する。

```text
1. VPC EndpointでPrivate DNSを有効にしているか
2. VPC側のDNS resolutionが有効か
3. VPC側のDNS hostnamesが有効か
4. 対象VPCを間違えていないか
5. 対象Subnetを間違えていないか
```

## Security Groupの考え方

Security Groupは「道」ではなく、通信の門番。

```text
Route Table
= 道案内・地図

VPC Endpoint
= AWSサービスへ行く専用窓口

Security Group
= 通信を通していいか判断する門番

IAM Role
= AWSサービスを利用していいかの許可証
```

今回のSecurity Group。

```text
private-ec2-sg
→ Private EC2に付ける

ssm-endpoint-sg
→ VPC Endpointに付ける
```

通信は以下。

```text
Private EC2
↓ HTTPS / TCP 443
VPC Endpoint
```

そのため、VPC Endpoint側のSecurity Groupに以下を設定した。

```text
Inbound:
HTTPS / TCP 443 / Source: private-ec2-sg
```

Private EC2側のInboundはなし。

SSH接続を使わないため、22番ポートは開けない。

## IAMとSecurity Groupの違い

IAMは、AWSサービスを操作・利用してよいかを決める。

```text
IAM
= 誰が・何が・どのAWSサービスに・何をしてよいか
```

Security Groupは、ネットワーク通信を通してよいかを決める。

```text
Security Group
= どの通信を通してよいか
```

例。

```text
IAM RoleでSSMを使う権限がある
でも通信経路がない
→ Systems Managerに届かない

Security Groupで443を許可している
でもIAM Roleがない
→ Systems Managerに受け付けてもらえない
```

つまり、権限と通信経路は別物。

## なぜSSHなしで接続できるのか

SSHでは、自分のPCからEC2へ直接接続する。

```text
自分のPC
↓
EC2の22番ポート
```

Session Managerでは、自分のPCからEC2へ直接接続しない。

```text
自分のブラウザ
↓
AWS Systems Manager
↓
SSM Agent
↓
EC2
```

EC2内のSSM AgentがSystems Managerと通信し、そのセッションをAWSコンソールから利用する。

そのため、以下が不要になる。

```text
Public IP
SSHポート開放
秘密鍵
踏み台サーバ
```

## 今回の成功条件

今回の接続成功に必要だったもの。

```text
1. EC2にSSM Agentがある
2. EC2に ec2-ssm-role が付いている
3. VPC Endpointが3つある
4. Endpoint用Security Groupで443が許可されている
5. Private DNSが有効
6. EC2のSecurity GroupでSSHを開けていない
7. EC2にPublic IPがない
```

## コマンドメモ

### whoami

```bash
whoami
```

説明。

現在のログインユーザーを表示する。

Session Managerで接続した場合、通常は以下のように表示される。

```text
ssm-user
```

### hostname

```bash
hostname
```

説明。

接続しているEC2のホスト名を表示する。

今どのサーバに入っているかを確認するために使う。

### ip a

```bash
ip a
```

説明。

ネットワークインターフェースとIPアドレスを確認する。

Private IPのみで動いていることを確認するために使う。

### curl -I

```bash
curl -I https://aws.amazon.com
```

説明。

指定したURLへHTTPリクエストを送り、レスポンスヘッダーだけを確認する。

今回はNAT GatewayもInternet Gatewayもないため、通常のインターネット通信は失敗する。

これにより、以下を確認できる。

```text
インターネット全体には出られない
でもVPC Endpoint経由でSystems Managerには接続できる
```

## トラブル時に見る場所

Session Managerに接続候補が出ない場合は、以下を確認する。

```text
1. EC2のStatus Checkが2/2になっているか
2. EC2にIAM Roleが付いているか
3. IAM Roleに AmazonSSMManagedInstanceCore が付いているか
4. VPC Endpoint 3つが Available になっているか
5. EndpointのSecurity Groupで443を許可しているか
6. VPCのDNS resolution / DNS hostnames が有効か
7. Private DNSが有効か
8. SSM Agentが入っているAMIか
```

## 削除時の注意

VPCを作成すると、以下が自動作成される。

```text
Main Route Table
Main Network ACL
Default Security Group
```

自分で作った覚えがないRoute Tableが表示されても、異常ではない。

VPC削除時に一緒に削除されるため、自分で作っていないMain Route Tableは基本的に放置でよい。

## 学び

今回の実践では、Private EC2へ接続する方法として、SSH以外の選択肢を学んだ。

特に重要だった理解。

```text
AWS Systems Manager
= EC2などを管理するAWSサービス

Session Manager
= Systems Managerの中にある、EC2へ接続する機能

SSM Agent
= EC2内で動くSystems Managerとの連絡係

IAM Role
= EC2がAWSサービスを使うための許可証

VPC Endpoint
= Private SubnetからAWSサービスへ行く専用窓口

Security Group
= 通信を通すか判断する門番
```

AWSでは、人間もEC2などのサービスも、AWSサービスを利用するには権限が必要。

```text
人間側の権限
→ 自分がAWSサービスを操作してよいか

EC2側の権限
→ EC2がAWSサービスを利用してよいか
```

今回のSSM接続では、以下がそろうことで接続できた。

```text
人間側の権限
+
EC2側のIAM Role
+
SSM Agent
+
VPC Endpoint
+
Security Group
+
Private DNS
```

## 一文まとめ

SSM Session Managerは、EC2へ直接SSHする仕組みではなく、EC2内のSSM AgentがAWS Systems Managerと通信し、そのセッションをAWSコンソールから利用する仕組み。NAT GatewayなしのPrivate Subnetでは、その通信経路をVPC Endpointで作る。
