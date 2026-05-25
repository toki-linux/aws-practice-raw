# AWS VPC Basic Lab - Cleanup

## 概要

このドキュメントは、自作VPC環境を削除する手順をまとめたものです。

AWS学習では、リソースを作成するだけでなく、安全に削除できることも重要です。

今回は以下のリソースを削除しました。

- EC2インスタンス
- セキュリティグループ
- サブネット
- ルートテーブル
- Internet Gateway
- VPC

---

## 削除の基本方針

削除は、VPC本体からではなく、依存しているリソースから行います。

考え方は以下です。

```text
中にあるもの
↓
接続しているもの
↓
最後に箱
```

VPCはネットワークの土台なので、最後に削除します。

---

## 今回の削除順

今回の削除順は以下です。

```text
1. EC2インスタンスを終了
2. 自作セキュリティグループを削除
3. サブネットを削除
4. ルートテーブルを削除
5. Internet Gatewayをデタッチして削除
6. VPCを削除
```

---

## 1. EC2インスタンスを終了する

まず、EC2インスタンスを終了します。

```text
EC2
→ Instances
→ 対象インスタンスを選択
→ Instance state
→ Terminate instance
```

ここでは `Stop` ではなく、`Terminate` を選びます。

```text
Stop      : 一時停止
Terminate : 削除
```

学習用の環境を完全に片付ける場合は、`Terminate` を使います。

---

## EC2を最初に削除する理由

EC2インスタンスは、以下のリソースに依存しています。

```text
EC2
├── Subnet
├── Security Group
├── Network Interface
└── EBS
```

EC2が残っていると、サブネットやセキュリティグループを削除できません。

そのため、最初にEC2を終了します。

---

## 2. セキュリティグループを削除する

EC2が終了したら、自分で作成したセキュリティグループを削除します。

```text
EC2
→ Security Groups
→ 自作セキュリティグループを選択
→ Actions
→ Delete security groups
```

注意点は、`default` セキュリティグループを削除しないことです。

今回削除するのは、自分で作成したセキュリティグループのみです。

```text
削除対象:
- toki-vpc-sg-01 など、自分で作成したもの

削除しない:
- default
```

---

## default セキュリティグループについて

VPCを新しく作成すると、そのVPC専用の `default` セキュリティグループが自動作成されます。

そのため、セキュリティグループ一覧に `default` が複数表示されることがあります。

これは異常ではありません。

```text
Default VPC
└── default Security Group

自作VPC
└── default Security Group
```

見分けるときは、セキュリティグループ名ではなく `VPC ID` を確認します。

自作VPCを削除すると、そのVPCに紐づく `default` セキュリティグループも一緒に削除されます。

---

## 3. サブネットを削除する

次に、作成したサブネットを削除します。

```text
VPC
→ Subnets
→ 自作サブネットを選択
→ Actions
→ Delete subnet
```

削除対象の例です。

```text
toki-public-subnet-01
```

サブネットが削除できない場合、まだ中にリソースが残っている可能性があります。

よくある原因は以下です。

```text
EC2が完全にterminatedになっていない
Network Interfaceが残っている
他のリソースがサブネットを使用している
```

その場合は、少し時間を置いてから再度削除します。

---

## 4. ルートテーブルを削除する

次に、作成したルートテーブルを削除します。

```text
VPC
→ Route tables
→ 自作ルートテーブルを選択
→ Actions
→ Delete route table
```

削除対象の例です。

```text
toki-public-rt-01
```

注意点は、`main` ルートテーブルを削除しないことです。

```text
削除対象:
- 自分で作成したルートテーブル

削除しない:
- main route table
```

---

## ルートテーブルが削除できない場合

ルートテーブルがサブネットに関連付いたままだと、削除できないことがあります。

その場合は、先にサブネットとの関連付けを外します。

```text
Route tables
→ 対象ルートテーブル
→ Subnet associations
→ Edit subnet associations
→ チェックを外す
→ Save
```

その後、再度ルートテーブルを削除します。

---

## 5. Internet Gatewayをデタッチして削除する

Internet Gatewayは、まずVPCからデタッチします。

```text
VPC
→ Internet gateways
→ 対象のInternet Gatewayを選択
→ Actions
→ Detach from VPC
```

デタッチ後、削除します。

```text
Actions
→ Delete internet gateway
```

削除対象の例です。

```text
toki-igw-01
```

Internet Gatewayは、VPCにアタッチされたままだと削除できません。

---

## 6. VPCを削除する

最後に、VPC本体を削除します。

```text
VPC
→ Your VPCs
→ 自作VPCを選択
→ Actions
→ Delete VPC
```

削除対象の例です。

```text
toki-vpc-01
```

VPCはネットワークの土台なので、最後に削除します。

---

## なぜVPCは最後なのか

VPCは、以下のリソースの土台です。

```text
VPC
├── Subnet
├── Route Table
├── Security Group
├── Network ACL
└── Internet Gateway
```

これらが残っている状態でVPCを削除しようとしても、依存関係があるため削除できないことがあります。

そのため、VPCは最後に削除します。

---

## 削除前チェックリスト

削除前に以下を確認します。

```text
EC2インスタンス: terminated
Elastic IP: 作成していない / 解放済み
NAT Gateway: 作成していない
Load Balancer: 作成していない
RDS: 作成していない
自作Security Group: 削除済み
自作Subnet: 削除済み
自作Route Table: 削除済み
Internet Gateway: デタッチ・削除済み
```

---

## 今回の構成で特に注意するもの

今回の構成では、NAT Gatewayを作成していません。

NAT Gatewayは課金が発生しやすいため、学習初期では作成しない方針にしました。

今回主に確認すべきリソースは以下です。

```text
EC2
Security Group
Subnet
Route Table
Internet Gateway
VPC
```

---

## 削除の考え方

削除は、階層の上から消すのではなく、依存されているものを最後に残す考え方です。

```text
使っているものを先に消す
使われていた入れ物を後で消す
```

VPCは多くのリソースから参照される土台なので、最後に削除します。

---

## 学び

今回の削除作業を通して、AWSリソースには依存関係があることを確認しました。

EC2はサブネットやセキュリティグループを使用しています。

サブネットやルートテーブル、Internet GatewayはVPCに所属しています。

そのため、VPCを最初に削除するのではなく、EC2などの中身から削除していく必要があります。

作成だけでなく、削除まで含めて実践することで、AWSリソース同士のつながりを理解しやすくなりました。
