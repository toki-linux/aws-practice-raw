# NAT GatewayでPrivate EC2から外部通信する

## 概要

このドキュメントは、Private Subnet内のEC2インスタンスからNAT Gateway経由でインターネットへ通信できるようにした手順と、発生した疑問・切り分けをまとめたものです。

Public SubnetとPrivate Subnetを分けた構成において、Private EC2はそのままではインターネットへ通信できません。

今回はNAT Gatewayを追加し、Private Route Tableを変更することで、Private EC2から外部サイトへアクセスできることを確認しました。

---

## 構成

```text
VPC: 10.0.0.0/16

├── Public Subnet: 10.0.1.0/24
│   └── public-ec2
│       └── Public IPあり
│
└── Private Subnet: 10.0.2.0/24
    └── private-ec2
        └── Public IPなし
```

NAT Gateway追加後の構成は以下です。

```text
VPC: 10.0.0.0/16

├── Public Subnet: 10.0.1.0/24
│   ├── public-ec2
│   └── Internet Gatewayへのルートあり
│
└── Private Subnet: 10.0.2.0/24
    └── private-ec2
        └── 0.0.0.0/0 → NAT Gateway
```

通信経路は以下です。

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

## 症状

Private EC2から外部サイトへ通信しようとしました。

```bash
curl -I https://aws.amazon.com
```

### コマンド説明

```text
curl
→ URLへHTTPリクエストを送る

-I
→ レスポンス本文ではなく、ヘッダー情報だけを表示する

https://aws.amazon.com
→ 外部通信確認用のURL
```

NAT Gatewayを設定する前は、この通信が失敗しました。

---

## 原因

Private Route Tableにインターネットへ出るルートがなかったことが原因です。

NAT Gateway作成前のPrivate Route Tableは以下です。

```text
Destination: 10.0.0.0/16
Target: local
```

このルートはVPC内通信のためのものです。

```text
10.0.0.0/16 → local
```

は、同じVPC内のEC2同士の通信には使えます。

しかし、外部サイトへ向かう通信には使えません。

外部サイトへ出るには、以下のようなデフォルトルートが必要です。

```text
0.0.0.0/0 → NAT Gateway
```

---

## 解決

NAT Gatewayを作成し、Private Route Tableに以下のルートを追加しました。

```text
Destination: 0.0.0.0/0
Target: NAT Gateway
```

これにより、Private EC2から外部へ向かう通信がNAT Gatewayへ送られるようになりました。

---

## NAT Gateway作成手順

AWSコンソールで以下のように進みました。

```text
VPC
→ NAT gateways
→ Create NAT gateway
```

設定内容は以下です。

```text
アベイラビリティーモード: リージョナル - 新規
接続タイプ: パブリック
Elastic IP割り当て方法: 自動
```

---

## サブネット選択欄がなかった件

NAT Gateway作成時、配置先サブネットを選ぶ欄が見当たりませんでした。

最初は、後からサブネットを指定するのかと思いました。

しかし、今回選択したのはリージョナルNAT Gatewayでした。

リージョナルNAT Gatewayでは、従来のゾーナルNAT GatewayのようにPublic Subnetを明示的に選びません。

```text
従来のゾーナルNAT Gateway
→ Public Subnetに配置する

今回のリージョナルNAT Gateway
→ VPC単位で作成する
→ サブネット指定欄がない
→ AWS側が裏側の配置を管理する
```

そのため、サブネット選択欄がなくても、Private Route Tableのターゲットに作成したNAT Gatewayを指定できれば進められました。

---

## Private Route Tableの編集

NAT Gateway作成後、Private Route Tableにルートを追加しました。

```text
VPC
→ Route Tables
→ private-rt
→ Routes
→ Edit routes
→ Add route
```

追加内容は以下です。

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

Private EC2にSSH接続した状態で、以下を実行しました。

```bash
curl -I https://aws.amazon.com
```

結果として、HTTPステータスコード `200` が返りました。

```text
HTTP/2 200
```

これにより、Private EC2からNAT Gateway経由で外部サイトへ通信できていると判断しました。

---

## 成功時の通信経路

成功時の通信経路は以下です。

```text
private-ec2
↓
Private Route Table
↓
0.0.0.0/0 → NAT Gateway
↓
NAT Gateway
↓
Internet Gateway
↓
Internet
```

---

## NAT Gatewayでできること

NAT Gatewayを使うと、Private EC2からインターネットへ外向き通信ができます。

```text
private-ec2 → Internet
```

これにより、以下のような操作が可能になります。

```bash
curl -I https://aws.amazon.com
```

```bash
sudo apt update
```

### コマンド説明

```text
sudo apt update
→ Ubuntuのパッケージ情報をインターネット上のリポジトリから取得する
→ Private EC2が外部通信できない場合は失敗する
```

---

## NAT Gatewayでできないこと

NAT Gatewayは、外部からPrivate EC2へ直接入るための入口ではありません。

```text
Internet → private-ec2
```

これはできません。

NAT Gatewayは、Private EC2から開始した通信を外へ出し、その戻り通信を返すための仕組みです。

```text
private-ec2 → Internet
Internet → private-ec2への戻り通信
```

はできます。

しかし、外部から新しくPrivate EC2へ接続を開始することはできません。

---

## 切り分け

今回の切り分けは以下です。

### 1. NATなしで失敗を確認

```bash
curl -I https://aws.amazon.com
```

Private EC2から外部通信できないことを確認しました。

---

### 2. Private Route Tableを確認

Private Route Tableに以下しかないことを確認しました。

```text
10.0.0.0/16 → local
```

外部通信に必要な以下のルートがありませんでした。

```text
0.0.0.0/0 → NAT Gateway
```

---

### 3. NAT Gatewayを作成

NAT Gatewayを作成しました。

今回のNAT GatewayはリージョナルNAT Gatewayだったため、サブネット選択欄はありませんでした。

---

### 4. Private Route TableにNAT Gatewayを指定

Private Route Tableに以下を追加しました。

```text
0.0.0.0/0 → NAT Gateway
```

---

### 5. curlで再確認

```bash
curl -I https://aws.amazon.com
```

`200` が返りました。

これにより、原因はSecurity Groupではなく、Private Route Tableに外向きルートがなかったことだと判断できました。

---

## 削除手順

NAT Gatewayは課金が発生するため、検証後すぐに削除しました。

削除手順は以下です。

---

### 1. Private Route TableからNAT Gatewayへのルートを削除

```text
VPC
→ Route Tables
→ private-rt
→ Routes
→ Edit routes
→ 0.0.0.0/0 → NAT Gateway を削除
→ Save changes
```

Private Route Tableを以下の状態に戻しました。

```text
10.0.0.0/16 → local
```

---

### 2. NAT Gatewayを削除

```text
VPC
→ NAT gateways
→ 対象NAT Gatewayを選択
→ Actions
→ Delete NAT gateway
```

NAT Gatewayの削除には少し時間がかかります。

削除中はElastic IPがまだ関連付いているように見えることがあります。

---

### 3. Elastic IPを確認

NAT Gateway削除後、Elastic IPが残っていないか確認しました。

```text
EC2
→ Elastic IPs
```

今回の環境では、NAT Gateway削除後にElastic IPが自動的に消えました。

そのため、追加の解放操作は不要でした。

ただし、Elastic IPが残っている場合は、手動で解放します。

```text
EC2
→ Elastic IPs
→ 対象EIPを選択
→ Actions
→ Release Elastic IP addresses
```

---

### 4. EC2とVPC関連リソースを削除

NAT GatewayとElastic IPの確認後、以下を削除しました。

```text
1. EC2 2台をTerminate
2. 自作Security Groupを削除
3. Public Subnet / Private Subnetを削除
4. Public Route Table / Private Route Tableを削除
5. Internet GatewayをDetachして削除
6. VPCを削除
```

---

## よくある注意点

### NAT Gatewayは課金される

NAT Gatewayは作成している時間に応じて課金されます。

そのため、学習用途では検証後すぐに削除します。

---

### Elastic IPの解放忘れに注意

NAT Gateway作成時にElastic IPが割り当てられることがあります。

NAT Gateway削除後にElastic IPが残っている場合、不要なら解放します。

---

### Private Route Tableのルート削除を忘れない

NAT Gatewayを削除する前に、Private Route TableからNAT Gatewayへのルートを削除しておくと整理しやすいです。

```text
0.0.0.0/0 → NAT Gateway
```

---

## 今回の学び

今回の実践で、以下を理解しました。

```text
Private EC2は、そのままだとインターネットへ出られない
Private Route Tableに外向きルートがないため通信に失敗する
NAT Gatewayを使うと、Private EC2から外部へ通信できる
Private Route Tableに0.0.0.0/0 → NAT Gatewayを追加する必要がある
NAT Gatewayは外からPrivate EC2へ入る入口ではない
NAT GatewayはPrivate EC2から開始した通信の戻り通信を返す
リージョナルNAT Gatewayではサブネット選択欄がない場合がある
NAT GatewayとElastic IPは削除・解放確認が必要
```

---

## 30秒説明

Private EC2からインターネットへ通信できるか確認しました。

最初はPrivate Route Tableに `10.0.0.0/16 → local` しかなく、外向きのルートがなかったため、`curl -I https://aws.amazon.com` は失敗しました。

その後、NAT Gatewayを作成し、Private Route Tableに `0.0.0.0/0 → NAT Gateway` を追加しました。

再度curlを実行するとHTTPステータスコード200が返り、Private EC2からNAT Gateway経由でインターネットへ通信できることを確認しました。

この実践を通して、NAT GatewayはPrivate EC2から外へ出るための出口であり、外部からPrivate EC2へ直接入る入口ではないことを理解しました。
