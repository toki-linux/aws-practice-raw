
# Elastic Load Balancingの種類と使い分け

## 概要

Elastic Load Balancingは、受け取った通信を複数の接続先へ分散するAWSサービス。

略して、`ELB` と呼ばれる。

```text
Elastic Load Balancing
=
受け取った通信を
複数のサーバなどへ振り分ける仕組み
```

---

# なぜLoad Balancerが必要なのか？

Webサーバが1台しかない場合を考える。

```text
ユーザー
↓
EC2 1台
```

アクセスが増えると、1台のEC2だけでは処理しきれなくなる可能性がある。

また、そのEC2が故障すると、Webサイトを表示できなくなる。

---

## Load Balancerを使う場合

```text
ユーザー
↓
Load Balancer
├── EC2-A
├── EC2-B
└── EC2-C
```

通信を複数のEC2へ振り分けられる。

---

## 主な役割

```text
複数の接続先へ通信を分散する
正常な接続先へ通信を送る
異常な接続先へ通信を送らない
アクセス増加に対応する
可用性を高める
```

---

# ELBの種類

Elastic Load Balancingには、以下の4種類がある。

```text
Application Load Balancer
Network Load Balancer
Gateway Load Balancer
Classic Load Balancer
```

略称は以下。

| 正式名称 | 略称 |
|---|---|
| Application Load Balancer | ALB |
| Network Load Balancer | NLB |
| Gateway Load Balancer | GWLB |
| Classic Load Balancer | CLB |

---

# Application Load Balancerとは？

## 概要

Application Load Balancerは、HTTP・HTTPS通信の振り分けに向くLoad Balancer。

略して、`ALB` と呼ばれる。

```text
ALB
=
Webアプリケーション向けのLoad Balancer
```

---

## 基本構成

```text
インターネット
↓ HTTP・HTTPS
ALB
├── EC2-A
├── EC2-B
└── EC2-C
```

---

## 特徴

ALBは、HTTP・HTTPSリクエストの内容を見て振り分けられる。

たとえば、URLパスに応じて接続先を変えられる。

```text
/users
↓
ユーザー管理用サーバ

/orders
↓
注文管理用サーバ
```

---

## 構成例

```text
ユーザー
↓
ALB
├── /users  → ユーザー管理サービス
└── /orders → 注文管理サービス
```

---

## 向いている場面

```text
Webサイト
Webアプリケーション
HTTP・HTTPS通信
URLパスで振り分けたい
ホスト名で振り分けたい
```

---

## 最初の覚え方

```text
ALB
→ Webアプリ向け
→ HTTP・HTTPS
→ リクエスト内容を見て細かく振り分ける
```

---

# Network Load Balancerとは？

## 概要

Network Load Balancerは、高速なネットワーク通信の処理に向くLoad Balancer。

略して、`NLB` と呼ばれる。

```text
NLB
=
大量のネットワーク通信を
高速に処理するLoad Balancer
```

---

## 基本構成

```text
大量の通信
↓
NLB
├── EC2-A
├── EC2-B
└── EC2-C
```

---

## 特徴

NLBは、TCP・UDP・TLSなどの通信を扱う。

ALBのようにURLパスなどを見るのではなく、ネットワーク通信を高速に振り分けることが得意。

---

## 向いている場面

```text
大量の通信を処理したい
低遅延が重要
TCPやUDP通信を扱いたい
固定IPアドレスを使いたい
```

---

## 最初の覚え方

```text
NLB
→ 高速重視
→ TCP・UDPなど
→ 大量通信に向く
```

---

# Gateway Load Balancerとは？

## 概要

Gateway Load Balancerは、ファイアウォールなどの仮想ネットワーク機器へ通信を振り分けるためのLoad Balancer。

略して、`GWLB` と呼ばれる。

```text
GWLB
=
仮想ファイアウォールなどへ
通信を振り分けるLoad Balancer
```

---

## 基本構成

```text
通信
↓
GWLB
├── 仮想ファイアウォールA
├── 仮想ファイアウォールB
└── 仮想ファイアウォールC
↓
アプリケーション
```

---

## なぜ必要なのか？

セキュリティ製品を1台だけ置くと、負荷が集中したり、障害時に通信できなくなったりする。

```text
通信
↓
仮想ファイアウォール1台
↓
アプリケーション
```

GWLBを使うと、複数の仮想ファイアウォールへ通信を分散できる。

---

## 向いている場面

```text
仮想ファイアウォールを複数配置したい
侵入検知・侵入防止システムを配置したい
セキュリティ製品を経由させたい
ネットワーク機器をスケールさせたい
```

---

## 最初の覚え方

```text
GWLB
→ Gateway
→ 通信経路上のネットワーク機器向け
→ ファイアウォールなどを並べる
```

---

# Classic Load Balancerとは？

## 概要

Classic Load Balancerは、以前から提供されている旧世代のLoad Balancer。

略して、`CLB` と呼ばれる。

```text
CLB
=
旧世代のLoad Balancer
```

---

## 基本的な考え方

CLBも、複数のEC2へ通信を分散する。

```text
ユーザー
↓
CLB
├── EC2-A
└── EC2-B
```

---

## 注意点

新しく構築する場合は、基本的に現在の用途に合うLoad Balancerを選ぶ。

```text
Webアプリ
→ ALB

高速なTCP・UDP通信
→ NLB

仮想ファイアウォールなど
→ GWLB
```

CLBは、既存システムで登場する旧世代のLoad Balancerとして覚える。

---

# 4種類の比較

| 種類 | 略称 | 主な用途 | 最初の覚え方 |
|---|---|---|---|
| Application Load Balancer | ALB | HTTP・HTTPSのWebアプリ | Webアプリ向け |
| Network Load Balancer | NLB | TCP・UDPなどの高速通信 | 高速・大量通信向け |
| Gateway Load Balancer | GWLB | 仮想ファイアウォールなど | ネットワーク機器向け |
| Classic Load Balancer | CLB | 旧世代の構成 | 以前のLoad Balancer |

---

# ALBとNLBの違い

## ALB

ALBは、HTTP・HTTPSリクエストの内容を理解できる。

```text
URLパス
ホスト名
HTTPヘッダー
```

などを見て振り分けられる。

```text
/users
→ サーバA

/orders
→ サーバB
```

---

## NLB

NLBは、ネットワーク通信を高速に振り分ける。

```text
TCP
UDP
TLS
```

などを扱う。

---

## 例え

### ALB

```text
受付担当
→ お客さんの用件を聞く
→ 内容に応じて担当部署へ案内する
```

### NLB

```text
高速な交通整理
→ 大量の車を素早く振り分ける
```

---

# ELBとAuto Scalingの関係

ELBとAuto Scalingは、よく一緒に使われる。

```text
ユーザー
↓
ALB
├── EC2-A
├── EC2-B
└── EC2-C
```

アクセスが増えたら、Auto ScalingでEC2を増やす。

```text
アクセス増加
↓
Auto Scaling
↓
EC2-Dを追加
↓
ALBが新しいEC2にも通信を振り分ける
```

---

# ヘルスチェックとは？

Load Balancerは、接続先が正常に動いているか確認する。

これをヘルスチェックと呼ぶ。

```text
Load Balancer
↓
「正常に動いていますか？」
↓
EC2
```

正常なEC2へは通信を送る。

異常なEC2へは通信を送らない。

```text
EC2-A
→ 正常
→ 通信を送る

EC2-B
→ 異常
→ 通信を送らない
```

---

# 試験向けの覚え方

```text
ALB
→ Application
→ Webアプリ
→ HTTP・HTTPS
→ URLなどを見て振り分ける

NLB
→ Network
→ 高速
→ 大量通信
→ TCP・UDPなど

GWLB
→ Gateway
→ 仮想ファイアウォールなど
→ ネットワーク機器向け

CLB
→ Classic
→ 旧世代
```

---

# 全体のまとめ

```text
Elastic Load Balancing
→ 通信を複数の接続先へ分散するサービス

ALB
→ Webアプリケーション向け
→ HTTP・HTTPS
→ URLパスなどで振り分ける

NLB
→ 高速なネットワーク通信向け
→ TCP・UDPなど
→ 大量通信・低遅延に向く

GWLB
→ 仮想ファイアウォールなどのネットワーク機器向け

CLB
→ 旧世代のLoad Balancer

ヘルスチェック
→ 接続先が正常か確認する

Auto Scaling
→ 必要に応じてEC2などを増減する
```

