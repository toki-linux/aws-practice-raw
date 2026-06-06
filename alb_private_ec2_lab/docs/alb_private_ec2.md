# ALBからPrivate EC2のnginxへアクセスする構成

## 1. 概要

Internet-facing Application Load Balancer（ALB）を入口にして、Private Subnet内のEC2へHTTP通信を転送する環境を構築した。

Private EC2にはPublic IPを付与せず、インターネットから直接アクセスできない状態にする。

```text
自分のブラウザ
↓ HTTP : 80
Internet-facing ALB
↓ Listener
Target Group
↓ HTTP : 80
Private EC2
↓
nginx
↓
HTMLを返す
```

管理用の接続にはSSHを使わず、AWS Systems Manager Session Managerを使用した。

```text
自分
↓
Session Manager
↓
SSM系Interface Endpoint
↓
Private EC2
```

---

# 2. 今回の目的

今回の実践では、次の内容を確認する。

```text
ELBとALBの違い
ALBをPublic Subnetに置く理由
ALB用に異なるAZのPublic Subnetが2つ必要な理由
Target Groupの役割
Listenerの役割
Health Checkの役割
ALBからPrivate EC2へ通信できる仕組み
Security Group参照による通信制御
Private EC2へnginxをインストールする方法
S3 Gateway EndpointとRoute Tableの関係
nginx停止時の502 Bad Gateway
```

---

# 3. ELBとALBの違い

## Elastic Load Balancing（ELB）

ELBは、AWSが提供するロードバランシング機能全体のサービス名。

```text
Elastic Load Balancing
├── Application Load Balancer
├── Network Load Balancer
├── Gateway Load Balancer
└── Classic Load Balancer
```

## Application Load Balancer（ALB）

ALBは、ELBの中にあるロードバランサーの種類の1つ。

主にWeb通信を扱う。

```text
HTTP
HTTPS
```

今回の構成では、ブラウザから届いたHTTP通信をPrivate EC2のnginxへ転送する。

```text
ブラウザ
↓ HTTP
ALB
↓ HTTP
Private EC2
```

---

# 4. ALBを使う理由

Public EC2へ直接アクセスさせる構成では、EC2自身をインターネットへ公開する。

```text
Internet
↓
Public EC2
```

今回は、EC2をPrivate Subnetへ置く。

```text
Internet
↓
ALB
↓
Private EC2
```

Private EC2にはPublic IPを付けない。

```text
Internet
×
Private EC2へ直接アクセス
```

外部へ公開する入口をALBに絞ることで、EC2を直接公開せずにWebページを表示できる。

---

# 5. 今回の完成構成

```text
VPC
├── Public Subnet-A
│   └── ALBノード
│
├── Public Subnet-C
│   └── ALBノード
│
└── Private Subnet-A
    └── Private EC2
        └── nginx
```

通信経路。

```text
ブラウザ
↓ HTTP : 80
Internet Gateway
↓
ALB
↓ HTTP : 80
Private EC2
↓
nginx
```

管理用経路。

```text
自分
↓
AWS Systems Manager Session Manager
↓
SSM系Interface Endpoint
↓
Private EC2
```

---

# 6. ALB用にPublic Subnetを2つ作る理由

ALB自体は1つだけ作成する。

ただし、1つのALBを異なるAZへまたがらせるため、異なるAZにPublic Subnetを2つ用意する。

```text
ALB：1つ
├── Public Subnet-A：ap-northeast-1a
└── Public Subnet-C：ap-northeast-1c
```

内部では、各AZにALB側のノードが作成される。

```text
利用者
↓
ALBのDNS名
↓
├── AZ-A側のALBノード
└── AZ-C側のALBノード
```

片方のAZで問題が発生しても、別のAZ側で通信を受けられるようにする。

```text
ALBを2つ作る
× 違う

1つのALBを複数AZへまたがらせる
○ 正しい
```

---

# 7. 作成したリソース

```text
VPC
Public Subnet × 2
Private Subnet × 1
Internet Gateway
Public Subnet用Route Table
Private Subnet用Route Table
ALB用Security Group
Private EC2用Security Group
SSM Endpoint用Security Group
SSM用IAM Role
SSM系Interface Endpoint × 3
S3 Gateway Endpoint
Private EC2
nginx
Target Group
Application Load Balancer
```

---

# 8. VPCとSubnetを作成

## 8.1 VPC

例。

```text
VPC CIDR：
10.0.0.0/16
```

VPCのDNS関連設定も確認する。

```text
DNS resolution：
有効

DNS hostnames：
有効
```

Interface EndpointのPrivate DNSを利用するために必要。

---

## 8.2 Subnet

異なるAZにPublic Subnetを2つ作成する。

```text
Public Subnet-A
AZ：ap-northeast-1a
CIDR：10.0.1.0/24
```

```text
Public Subnet-C
AZ：ap-northeast-1c
CIDR：10.0.2.0/24
```

Private EC2用のPrivate Subnetを1つ作成する。

```text
Private Subnet-A
AZ：ap-northeast-1a
CIDR：10.0.3.0/24
```

---

# 9. Public Subnet用Route Table

Internet Gatewayを作成し、VPCへアタッチする。

Public Subnet用Route Tableに、次のルートを追加する。

```text
Destination：0.0.0.0/0
Target：Internet Gateway
```

意味。

```text
ほかの具体的なルートに一致しなかった通信
↓
Internet Gatewayへ送る
```

Public Subnet-AとPublic Subnet-Cを、このRoute Tableへ関連付ける。

```text
Public Subnet-A
Public Subnet-C
↓
public-route-table
├── 10.0.0.0/16 → local
└── 0.0.0.0/0  → Internet Gateway
```

Private Subnetは、このRoute Tableへ関連付けない。

---

# 10. Security Groupの設計

## 10.1 ALB用Security Group

名前の例。

```text
alb-sg
```

Inbound Rule。

```text
Type：HTTP
Protocol：TCP
Port：80
Source：自分のIP
```

学習用として、自分のブラウザからだけALBへアクセスできるようにした。

```text
自分のIP
↓ HTTP : 80
ALB
```

---

## 10.2 Private EC2用Security Group

名前の例。

```text
private-sg
```

Inbound Rule。

```text
Type：HTTP
Protocol：TCP
Port：80
Source：alb-sg
```

Private EC2は、ALBからのHTTP通信だけを受け付ける。

```text
ALB
└── alb-sg
        ↓ HTTP : 80
Private EC2
└── private-sg
```

EC2にはSSH接続用のTCP `22` を開けない。

```text
SSH : 22
→ 不要
```

---

## 10.3 SSM Endpoint用Security Group

名前の例。

```text
ssm-sg
```

Inbound Rule。

```text
Type：HTTPS
Protocol：TCP
Port：443
Source：private-sg
```

Private EC2からSSM系Interface EndpointへHTTPS通信できるようにする。

---

# 11. IAM Role

Private EC2へ付与するIAM Roleに、次のAWS管理ポリシーを付ける。

```text
AmazonSSMManagedInstanceCore
```

目的。

```text
EC2内のSSM Agent
↓
AWS Systems Managerと通信
↓
Session Managerで接続
```

これは、ALBからEC2へHTTP通信するための権限ではない。

```text
ALB → EC2のHTTP通信
→ IAM Role不要
→ Security Groupで許可する

Session Manager → EC2への管理接続
→ IAM Role必要
```

---

# 12. SSM系Interface Endpoint

Private EC2にはPublic IPを付けず、NAT Gatewayも作らない。

Session ManagerでPrivate EC2へ接続するため、次のInterface Endpointを作成する。

```text
com.amazonaws.ap-northeast-1.ssm

com.amazonaws.ap-northeast-1.ssmmessages

com.amazonaws.ap-northeast-1.ec2messages
```

タイプ。

```text
Interface
```

設定。

```text
VPC：
今回作成したVPC

Subnet：
Private Subnet-A

Security Group：
ssm-sg

Private DNS：
有効
```

---

# 13. Private EC2を作成

Private Subnet-AへEC2を作成する。

```text
Subnet：
Private Subnet-A

Public IP：
なし

Security Group：
private-sg

IAM Role：
AmazonSSMManagedInstanceCoreを含むRole
```

Session Managerで接続できることを確認する。

```text
EC2
↓
接続
↓
Session Manager
↓
接続
```

---

# 14. S3 Gateway Endpoint

## 14.1 なぜ必要か

Private EC2へnginxをインストールするため、次のコマンドを実行した。

```bash
sudo dnf install nginx
```

しかし、Private EC2には次の経路がない。

```text
Public IPなし
NAT Gatewayなし
Internet Gateway経由の外向き通信なし
```

Amazon Linux 2023のパッケージ取得先は、AWS管理のS3バケット上にある。

```text
Private EC2
↓
Amazon Linuxのパッケージリポジトリ
↓
S3上のパッケージ置き場
```

そのため、Private EC2からS3へ到達するための専用経路を作る。

サービス名。

```text
com.amazonaws.ap-northeast-1.s3
```

タイプ。

```text
Gateway
```

---

## 14.2 Route Tableとの関係

S3 Gateway Endpointを作成するときは、Private EC2が使っているRoute Tableへ関連付ける。

```text
Private Subnet-A
↓
private-route-table
```

正しい状態。

```text
private-route-table
├── 10.0.0.0/16 → local
└── pl-xxxxxxxx  → S3 Gateway Endpoint
```

`pl-xxxxxxxx` はPrefix List。

```text
S3が利用するIPアドレス範囲のまとまり
```

---

# 15. nginxをインストール

S3 Gateway Endpointを追加後、Private EC2で実行。

```bash
sudo dnf install nginx
```

コマンドの意味。

```text
dnf
→ Amazon Linuxでパッケージを管理するコマンド

install nginx
→ nginxパッケージを取得してインストール
```

nginxを起動し、OS再起動後も自動起動するように設定する。

```bash
sudo systemctl enable --now nginx
```

意味。

```text
systemctl
→ Linuxのサービスを操作する

enable
→ OS起動時にnginxも自動起動する

--now
→ 自動起動設定と同時に、今すぐ起動する
```

---

# 16. HTMLファイルを配置

Amazon Linux 2023では、nginxのドキュメントルートがUbuntuと異なった。

Ubuntuでよく使う場所。

```text
/var/www/html
```

Amazon Linux 2023で使用した場所。

```text
/usr/share/nginx/html
```

テスト用HTMLを作成する。

```bash
echo "hello from private ec2" | sudo tee /usr/share/nginx/html/index.html
```

意味。

```text
echo "hello from private ec2"
→ 文字列を出力

|
→ 左側の出力を右側へ渡す

sudo tee /usr/share/nginx/html/index.html
→ root権限でindex.htmlへ書き込む
```

EC2内部から確認。

```bash
curl http://localhost
```

意味。

```text
curl
→ HTTPリクエストを送る

localhost
→ 今操作しているEC2自身
```

期待する結果。

```text
hello from private ec2
```

---

# 17. Target Groupを作成

Target Groupは、ALBが通信を送る相手の一覧。

```text
ALB
↓
Target Group
└── Private EC2
```

設定。

```text
Target type：
Instances

Protocol：
HTTP

Port：
80

VPC：
今回作成したVPC
```

Health Check。

```text
Protocol：
HTTP

Path：
/
```

Private EC2をターゲットとして登録する。

登録画面では、EC2を選択したあとに、

```text
保留中として以下に含める
```

を押し、保留中ターゲット一覧へ追加する。

その後、登録を確定する。

---

# 18. Target Groupの役割

Target Groupは、ALBが通信を送る相手とポート番号を管理する。

```text
Target Group
├── Protocol：HTTP
├── Port：80
├── Health Check：HTTP /
└── Target：Private EC2
```

Target Groupを作るだけでは、通信は完成しない。

必要なもの。

```text
① ALBのListenerがHTTP 80で待ち受ける
② Listener RuleがTarget Groupを選ぶ
③ Target GroupへPrivate EC2を登録する
④ private-sgでalb-sgからのHTTP 80を許可する
```

---

# 19. ALBを作成

ロードバランサー作成画面で、Application Load Balancerを選択する。

設定。

```text
Load balancer type：
Application Load Balancer

Scheme：
Internet-facing

IP address type：
IPv4

VPC：
今回作成したVPC
```

Subnet。

```text
Public Subnet-A
Public Subnet-C
```

異なるAZのPublic Subnetを2つ選択する。

Security Group。

```text
alb-sg
```

Listener。

```text
Protocol：
HTTP

Port：
80

Forward to：
作成済みTarget Group
```

---

# 20. 今回使用しなかったオプション

ALB作成画面には、次の追加機能も表示された。

```text
CloudFront + WAF
AWS WAF
AWS Global Accelerator
IPAMプール
Target Group Stickiness
複数Target Groupの重み付け
```

今回は基本構成を理解することが目的なので、使用しなかった。

```text
CloudFront + WAF
→ チェックしない

AWS WAF
→ チェックしない

AWS Global Accelerator
→ チェックしない

IPAMプール
→ 使用しない

Target Group Stickiness
→ 使用しない

Target Groupの重み
→ Target Groupが1つなので初期値のまま
```

---

# 21. Listenerとは

Listenerは、ALBが受け取る通信を待ち受ける設定。

今回。

```text
Listener
└── HTTP : 80
        ↓
    Target Groupへ転送
```

通信の流れ。

```text
ブラウザ
↓ HTTP : 80
ALB
↓ Listener
Target Group
↓ HTTP : 80
Private EC2
```

---

# 22. Health Checkとは

ALBは、Target Groupへ登録されたEC2が正常に応答できるかを定期的に確認する。

今回。

```text
ALB
↓ HTTP GET /
Private EC2
↓
nginxが200 OKを返す
```

正常時。

```text
healthy
```

異常時。

```text
unhealthy
```

ALBは、正常なターゲットへ通信を送る。

---

# 23. ブラウザで動作確認

ALB作成後、ALBの状態が次のように変わるまで待つ。

```text
プロビジョニング中
↓
アクティブ
```

Target Groupも確認する。

```text
initial
↓
healthy
```

ALBのDNS名を確認する。

例。

```text
toki-alb-xxxxxxxx.ap-northeast-1.elb.amazonaws.com
```

ブラウザからアクセスする。

```text
http://ALBのDNS名
```

期待する表示。

```text
hello from private ec2
```

---

# 24. トラブルシューティング

## 24.1 症状：nginxをインストールできない

実行したコマンド。

```bash
sudo dnf install nginx
```

しかし、処理がなかなか進まなかった。

---

## 24.2 原因

Private Subnetが使用するRoute Tableと、S3 Gateway Endpointへ関連付けたRoute Tableが異なっていた。

誤った状態。

```text
Public Subnet-A
Public Subnet-C
↓
public-route-table
├── 10.0.0.0/16 → local
├── 0.0.0.0/0  → Internet Gateway
└── pl-S3       → S3 Gateway Endpoint
```

```text
Private Subnet-A
↓
private-route-table
└── 10.0.0.0/16 → local
```

Private EC2は、自分が所属するSubnetに関連付いたRoute Tableを見る。

```text
Private EC2
↓
Private Subnet-A
↓
private-route-table
```

Private EC2が使用するRoute Tableに、S3向けルートが存在しなかった。

```text
Private EC2
↓
private-route-tableを見る
↓
S3向けルートなし
↓
S3上のパッケージ置き場へ到達できない
↓
nginxを取得できない
```

---

## 24.3 解決

S3 Gateway Endpointを、Private Subnet用Route Tableへ関連付ける。

修正後。

```text
private-route-table
├── 10.0.0.0/16 → local
└── pl-S3       → S3 Gateway Endpoint
```

通信経路。

```text
Private EC2
↓
Private Subnet用Route Table
↓
S3 Gateway Endpoint
↓
S3上のAmazon Linuxパッケージ置き場
↓
nginxを取得
```

再度実行。

```bash
sudo dnf install nginx
```

インストール成功。

---

## 24.4 切り分け

今回の切り分け。

```text
Session ManagerでPrivate EC2へ接続できる
↓
SSM系Interface Endpointは正常

dnf install nginxが進まない
↓
パッケージ取得先への経路を確認

Private EC2はPublic IPなし
NAT Gatewayなし
↓
S3 Gateway Endpointを確認

Private EC2用Route TableにS3向けルートなし
↓
Route Tableの関連付けミス
```

重要。

```text
Private Subnetだけ別のRoute Tableを使う
→ 問題ではない

Private EC2が使うRoute Tableに
S3 Gateway Endpoint向けルートがない
→ 問題
```

---

# 25. トラブルシューティング実践：nginx停止

動作確認後、意図的にnginxを停止した。

Private EC2で実行。

```bash
sudo systemctl stop nginx
```

意味。

```text
nginxサービスを停止する
```

ブラウザでALBのDNS名へアクセス。

結果。

```text
502 Bad Gateway
```

---

# 26. 502 Bad Gatewayの意味

今回の502は、次の状態を表す。

```text
ブラウザ
↓
ALBまでは到達できた
↓
Private EC2へ転送
↓
nginxが停止している
↓
正常な応答を受け取れない
↓
502 Bad Gateway
```

つまり、

```text
ALBへの通信は成功
転送先のnginxから正常な応答がない
```

という意味。

Target Groupの状態も、時間が経つと変化する。

```text
healthy
↓
unhealthy
```

---

# 27. nginxを復旧

Private EC2で実行。

```bash
sudo systemctl start nginx
```

意味。

```text
nginxサービスを起動する
```

EC2内部から確認。

```bash
curl http://localhost
```

期待する結果。

```text
hello from private ec2
```

少し待つと、Target Groupも復旧する。

```text
unhealthy
↓
healthy
```

ブラウザでALBのDNS名へ再アクセスし、Webページが表示されることを確認した。

---

# 28. ALBとPrivate EC2はどうつながるか

ALBとEC2は、ケーブルのように直接つなぐわけではない。

```text
ALB
↓
Listener
↓
Target Group
↓
Private EC2のPrivate IP : 80
```

Target GroupへEC2を登録すると、ALBはEC2のPrivate IP宛てに通信を送る。

EC2にPublic IPは不要。

```text
ALB
↓ VPC内部通信
Private EC2のPrivate IP : 80
```

IAM Roleも不要。

```text
ALB → EC2
→ 通常のHTTP通信
→ Security Groupで制御する
```

---

# 29. S3 Gateway EndpointとRDS用Endpointの違い

前回のRDS実践でも、S3 Gateway Endpointを使用した。

今回も役割は同じ。

```text
S3 Gateway Endpoint
→ Private EC2からS3上のパッケージ置き場へ到達する道
```

RDS接続用のEndpointではない。

```text
RDSを使うからS3が必要
× 違う

Private EC2へソフトをインストールするため、
S3上のパッケージを取得する必要があった
○ 正しい
```

---

# 30. Route Tableの理解

Subnetは、自分に関連付いたRoute Tableを使う。

```text
Subnet
↓
関連付いたRoute Table
↓
宛先に応じて次の行き先を決める
```

Private EC2が複数のRoute Tableを自由に見比べるわけではない。

```text
Private EC2
↓
所属するPrivate Subnet
↓
Private Subnetに関連付いたRoute Table
```

PublicとPrivateでRoute Tableを分けるのは正常。

```text
public-route-table
├── 10.0.0.0/16 → local
└── 0.0.0.0/0  → Internet Gateway
```

```text
private-route-table
├── 10.0.0.0/16 → local
└── pl-S3       → S3 Gateway Endpoint
```

---

# 31. 削除手順

学習終了後、課金を避けるためにリソースを削除した。

削除順序。

```text
① ALB
② Target Group
③ Private EC2
④ VPC Endpoint
⑤ Security Group
⑥ Subnet
⑦ Route Table
⑧ Internet Gateway
⑨ VPC
⑩ IAM Role
```

---

## 31.1 ALBを削除

```text
EC2
↓
ロードバランシング
↓
ロードバランサー
↓
作成したALBを選択
↓
アクション
↓
ロードバランサーを削除
```

ALBは課金対象なので、最初に削除する。

---

## 31.2 Target Groupを削除

```text
EC2
↓
ロードバランシング
↓
ターゲットグループ
↓
作成したTarget Groupを選択
↓
アクション
↓
削除
```

ALBに関連付いた状態では削除できないため、ALBを先に消す。

---

## 31.3 Private EC2を終了

```text
EC2
↓
インスタンス
↓
Private EC2を選択
↓
インスタンスの状態
↓
インスタンスを終了
```

---

## 31.4 VPC Endpointを削除

今回作成したEndpoint。

```text
com.amazonaws.ap-northeast-1.ssm

com.amazonaws.ap-northeast-1.ssmmessages

com.amazonaws.ap-northeast-1.ec2messages

com.amazonaws.ap-northeast-1.s3
```

削除場所。

```text
VPC
↓
エンドポイント
↓
対象Endpointを選択
↓
アクション
↓
VPCエンドポイントを削除
```

---

## 31.5 Security Groupを削除

削除順。

```text
alb-sg
↓
ssm-sg
↓
private-sg
```

`private-sg` は、`alb-sg` や `ssm-sg` から参照されているため、最後に削除する。

---

## 31.6 Subnetを削除

今回作成したSubnet。

```text
Public Subnet-A
Public Subnet-C
Private Subnet-A
```

---

## 31.7 Route Tableを削除

自分で作成したRoute Tableを削除する。

```text
public-route-table
private-route-table
```

メインRoute Tableは、VPC削除時に一緒に削除される。

---

## 31.8 Internet Gatewayを削除

まずVPCからデタッチする。

```text
VPC
↓
インターネットゲートウェイ
↓
対象IGWを選択
↓
アクション
↓
VPCからデタッチ
```

その後、削除する。

---

## 31.9 VPCを削除

```text
VPC
↓
お使いのVPC
↓
対象VPCを選択
↓
アクション
↓
VPCを削除
```

---

## 31.10 IAM Roleを削除

今回専用に作成したSSM用IAM Roleを削除する。

今後の学習で使い回す場合は、残してもよい。

---

# 32. 今回の学び

```text
ELB
→ AWSのロードバランシング機能全体

ALB
→ ELBの種類の1つ
→ HTTP・HTTPSなどのWeb通信を扱う

ALB
→ インターネット側の入口

Listener
→ ALBが受け取った通信をどこへ送るか決める

Target Group
→ ALBが通信を送る相手の一覧

Health Check
→ EC2が正常に応答するか定期確認する

Private EC2
→ ALBからのHTTP通信だけを受け取る

Security Group参照
→ 相手のSGをSourceに指定して通信を制御する

S3 Gateway Endpoint
→ Private EC2がS3上のパッケージ置き場へ行くための道

Route Table
→ Subnetごとに通信の次の行き先を決める

502 Bad Gateway
→ ALBには到達したが、
   転送先から正常な応答を受け取れなかった
```

---

# 33. 構築結果

以下を自力で構築・確認・削除できた。

```text
VPC
Public Subnet × 2
Private Subnet × 1
Internet Gateway
Public Subnet用Route Table
Private Subnet用Route Table
ALB用Security Group
Private EC2用Security Group
SSM Endpoint用Security Group
SSM用IAM Role
SSM系Interface Endpoint
S3 Gateway Endpoint
Private EC2
Session Manager接続
nginxインストール
HTML配置
Target Group
Health Check
Application Load Balancer
ALBのDNS名からWebページ表示
nginx停止による502エラー再現
nginx再起動による復旧
全リソース削除
```

---

# 34. 今回の要点

```text
ブラウザ
↓
Internet-facing ALB
↓
Listener
↓
Target Group
↓
Private EC2
↓
nginx
```

Private EC2をインターネットへ直接公開せず、ALBだけを入口にしてWebページを表示できた。

また、nginx停止時に502エラーを再現し、ALBが転送先の状態に影響を受けることを確認できた。

