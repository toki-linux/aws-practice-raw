# SSH接続できなかった原因：ルートテーブルのサブネット関連付け漏れ

## 概要

自作VPC内にEC2インスタンスを作成したあと、SSH接続を試したが接続できなかった。

最終的な原因は、ルートテーブルに `0.0.0.0/0 → Internet Gateway` のルートは追加していたものの、そのルートテーブルをEC2が所属するサブネットに関連付けていなかったことだった。

つまり、ルートテーブルの中身は正しかったが、対象サブネットがそのルートテーブルを使っていない状態だった。

---

## 発生した問題

自作VPC内にEC2インスタンスを作成し、SSH接続しようとした。

```bash
ssh -i key.pem ubuntu@<EC2のPublic IPv4>
```

しかし、SSH接続できなかった。

---

## 作成していた構成

今回作成していた構成は以下。

```text
VPC
└── Public Subnet
    └── EC2 Instance
        ├── Public IPv4 Address
        └── Security Group
```

インターネット接続用に以下も作成していた。

```text
Internet Gateway
Route Table
Security Group
```

ルートテーブルには、以下のルートを追加していた。

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

そのため、一見すると外部通信できる設定になっているように見えた。

---

## 原因

原因は、ルートテーブルの **Subnet associations** で、EC2が所属するサブネットを関連付けていなかったこと。

やっていたことは以下。

```text
Routes タブ
0.0.0.0/0 → igw-xxxx
を追加した
```

しかし、できていなかったことは以下。

```text
Subnet associations タブ
対象サブネットにチェックを入れていなかった
```

つまり、ルートテーブルという「地図」には外への道を書いていたが、その地図をサブネットに持たせていなかった。

---

## イメージ

```text
Internet Gateway = 外への出口

Route Table = 外への道が書かれた地図

Subnet Association = このサブネットはこの地図を使います、という設定
```

今回の状態は以下。

```text
地図は完成している
でも
その地図をサブネットが使っていない
```

そのため、EC2が所属するサブネットから見ると、Internet Gatewayへの道がない状態だった。

---

## 正しい状態

正しい状態は以下。

```text
Route Table
├── Routes
│   ├── 10.0.0.0/16 → local
│   └── 0.0.0.0/0  → Internet Gateway
│
└── Subnet associations
    └── EC2が所属するPublic Subnet
```

重要なのは、以下の2つをセットで確認すること。

```text
1. Routes に 0.0.0.0/0 → Internet Gateway がある
2. Subnet associations に EC2のサブネットが関連付いている
```

---

## 解決方法

VPCのルートテーブル画面で、対象のルートテーブルを開いた。

その後、以下を実施した。

```text
Route tables
→ 対象のルートテーブルを選択
→ Subnet associations
→ Edit subnet associations
→ EC2が所属するサブネットにチェック
→ Save associations
```

これにより、EC2が所属するサブネットが、Internet Gatewayへのルートを持つルートテーブルを使うようになった。

その後、SSH接続できるようになった。

---

## 確認したこと

SSH接続できない原因として、以下を確認した。

```text
EC2にPublic IPv4があるか
Security GroupでSSHが許可されているか
EC2が自作VPC・自作サブネットに所属しているか
Internet GatewayがVPCにアタッチされているか
Route Tableに 0.0.0.0/0 → IGW があるか
Route Tableが対象サブネットに関連付いているか
SSHユーザー名が正しいか
キーペアが正しいか
```

今回の原因は、この中の以下だった。

```text
Route Tableが対象サブネットに関連付いていなかった
```

---

## なぜSSH接続できなかったのか

外部のPCからEC2へSSH接続するとき、通信はEC2のPublic IP宛てに届く。

しかし、SSH通信は一方通行ではなく、EC2から自分のPCへ返事を返す必要がある。

```text
自分のPC
↓
EC2

EC2
↓
自分のPC
```

EC2から自分のPCへ返事を返す通信は、EC2から見ると外向き通信になる。

そのため、サブネットのルートテーブルに以下のルートが必要。

```text
0.0.0.0/0 → Internet Gateway
```

今回はこのルート自体は作成していた。

しかし、そのルートテーブルをEC2のサブネットに関連付けていなかったため、EC2は外へ返事を返せなかった。

結果として、SSH接続が成立しなかった。

---

## 今回の失敗ポイント

今回の失敗ポイントは、以下の2つを同じ作業だと思い込んでいたこと。

```text
ルートを追加する
サブネットに関連付ける
```

実際には、これは別作業。

```text
Routes タブ
→ ルートテーブルの中に道を書く作業

Subnet associations タブ
→ どのサブネットがそのルートテーブルを使うか決める作業
```

---

## 次回からの確認手順

次回からは、SSH接続できない場合、以下の順で確認する。

```text
1. EC2の詳細画面で Subnet ID を確認する
2. VPC → Route Tables を開く
3. 対象のルートテーブルを開く
4. Routes に 0.0.0.0/0 → Internet Gateway があるか確認する
5. Subnet associations に EC2のSubnet ID があるか確認する
```

特に以下をセットで見る。

```text
Routes
Subnet associations
```

---

## 学び

今回のSSH接続失敗の原因は、ルートテーブルに `0.0.0.0/0 → Internet Gateway` は追加していたものの、そのルートテーブルをEC2が所属するサブネットに関連付けていなかったことだった。

この経験から、パブリックサブネットとして機能させるには、ルートを追加するだけでなく、対象サブネットとの関連付けまで確認する必要があると学んだ。

```text
ルートを書く
＋
サブネットに関連付ける
＝
そのサブネットが外への道を使える
```

AWSでは「設定したつもり」と「実際に関連付いている」は別物だとわかった。

---

## 30秒説明

自作VPC内のEC2にSSH接続できない問題が発生しました。

原因は、ルートテーブルに `0.0.0.0/0 → Internet Gateway` のルートは追加していたものの、そのルートテーブルをEC2が所属するサブネットに関連付けていなかったことです。

ルートテーブルに道を書くだけでは、そのサブネットがその道を使うとは限りません。

`Subnet associations` で対象サブネットを関連付けたところ、SSH接続できるようになりました。

この経験から、パブリックサブネットにするには、Internet Gatewayへのルート追加と、サブネット関連付けの両方が必要だと理解しました。
