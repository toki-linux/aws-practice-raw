# 構築手順

## 1. ゴール

Private Subnet に EC2 を作成し、Session Manager で接続する。

その後、S3 Gateway Endpoint を経由して、S3 との間でファイルを送受信する。

```text
Private EC2
↓
S3 Gateway Endpoint
↓
S3
```

---

## 2. 前提

今回の EC2 は、インターネットへ直接接続しない。

```text
Public IP: なし
SSH: 使用しない
NAT Gateway: 使用しない
Internet Gateway向けルート: 作成しない
```

EC2 への接続には Session Manager を使う。

---

## 3. IAM Role を作成する

EC2 用 IAM Role を作成する。

例:

```text
toki-private-ec2-s3-role
```

以下のポリシーを付与する。

```text
AmazonSSMManagedInstanceCore
AmazonS3FullAccess
```

役割:

```text
AmazonSSMManagedInstanceCore
→ SSM Agent が Systems Manager と通信するため

AmazonS3FullAccess
→ EC2 内の AWS CLI から S3 を読み書きするため
```

注意:

```text
AmazonS3FullAccess
```

は権限が広い。
今回は学習用として使用する。

---

## 4. VPC を作成する

例:

```text
Name: toki-private-ec2-s3-vpc
IPv4 CIDR: 10.0.0.0/16
```

VPC の DNS 関連設定を確認する。

```text
DNS resolution: 有効
DNS hostnames: 有効
```

Interface Endpoint の Private DNS を利用するために必要。

---

## 5. Private Subnet を作成する

例:

```text
Name: toki-private-subnet
IPv4 CIDR: 10.0.1.0/24
```

今回、この Subnet に EC2 と Interface Endpoint を配置する。

---

## 6. Private Subnet 用 Route Table を作成する

例:

```text
Name: toki-private-route-table
```

Private Subnet と関連付ける。

最初は local ルートのみ。

```text
Destination: 10.0.0.0/16
Target: local
```

今回は、以下のような外向きルートを追加しない。

```text
0.0.0.0/0 → Internet Gateway
0.0.0.0/0 → NAT Gateway
```

---

## 7. Private EC2 用 Security Group を作成する

例:

```text
Name: toki-private-ec2-sg
```

設定:

```text
Inbound:
なし

Outbound:
すべて許可
```

SSH を使用しないため、TCP 22 番は開けない。

---

## 8. Interface Endpoint 用 Security Group を作成する

例:

```text
Name: toki-endpoint-sg
```

設定:

```text
Inbound:
HTTPS / TCP 443
Source: toki-private-ec2-sg

Outbound:
すべて許可
```

意味:

```text
Private EC2用SGが付いたEC2から、
Interface EndpointのENIへ届くHTTPS通信を許可する
```

Security Group 自体が通信するわけではない。
実際に通信する主体は EC2。

---

## 9. SSM 系 Interface Endpoint を作成する

以下の3つを作成する。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

タイプ:

```text
Interface
```

共通設定:

```text
VPC: toki-private-ec2-s3-vpc
Subnet: toki-private-subnet
Security Group: toki-endpoint-sg
Private DNS: 有効
```

---

## 10. S3 Gateway Endpoint を作成する

サービス名:

```text
com.amazonaws.ap-northeast-1.s3
```

タイプ:

```text
Gateway
```

関連付ける Route Table:

```text
toki-private-route-table
```

Gateway Endpoint を作成すると、Route Table に S3 向けルートが追加される。

例:

```text
Destination: pl-xxxxxxxx
Target: vpce-xxxxxxxx
```

`pl-xxxxxxxx` は Prefix List。

```text
S3 が使用する IP アドレス範囲のまとまり
```

と考えればよい。

---

## 11. EC2 を作成する

AMI:

```text
Amazon Linux 2023
```

設定例:

```text
Subnet: toki-private-subnet
Public IP: 無効
IAM Role: toki-private-ec2-s3-role
Security Group: toki-private-ec2-sg
```

Amazon Linux 2023 を使用した理由:

```text
AWS CLI v2 が標準で利用できるため
```

---

## 12. S3 バケットを作成する

S3 コンソールでバケットを作成する。

例:

```text
toki-private-ec2-s3-lab-任意の文字列
```

S3 バケット名はグローバルで一意にする必要がある。

EC2 や VPC Endpoint と同じリージョンを選ぶ。

今回の例:

```text
ap-northeast-1
```

テスト用として、以下のファイルをアップロードする。

```text
test.txt
```

内容例:

```text
hello from s3
```

---

## 13. Session Manager で EC2 に接続する

EC2 コンソールで対象インスタンスを選択する。

```text
接続
↓
Session Manager
↓
接続
```

プロンプトが返れば接続成功。

---

## 14. AWS CLI を確認する

```bash
aws --version
```

説明:

```text
AWS CLI がインストールされているか確認する
```

Amazon Linux 2023 では、AWS CLI v2 が標準で利用できる。

---

## 15. IAM Role を確認する

```bash
aws sts get-caller-identity
```

説明:

```text
EC2 がどの IAM Role の権限を利用しているか確認する
```

---

## 16. S3 バケット一覧を確認する

```bash
aws s3 ls
```

説明:

```text
アクセス可能な S3 バケットの一覧を表示する
```

---

## 17. 対象バケットの中身を確認する

```bash
aws s3 ls s3://バケット名/
```

説明:

```text
指定した S3 バケット内のオブジェクト一覧を表示する
```

---

## 18. 書き込み可能なディレクトリへ移動する

```bash
cd ~
```

説明:

```text
現在のユーザーのホームディレクトリへ移動する
```

現在地を確認する。

```bash
pwd
```

説明:

```text
現在いるディレクトリを表示する
```

---

## 19. S3 から EC2 へファイルをダウンロードする

```bash
aws s3 cp s3://バケット名/test.txt .
```

説明:

```text
S3 上の test.txt を、
現在いるディレクトリへコピーする
```

最後の `.` は、現在のディレクトリを意味する。

ファイルを確認する。

```bash
ls -l
```

説明:

```text
現在のディレクトリ内にあるファイルを詳しく表示する
```

内容を確認する。

```bash
cat test.txt
```

説明:

```text
test.txt の中身を表示する
```

---

## 20. EC2 内でテストファイルを作成する

```bash
echo "hello from EC2" > ec2_test.txt
```

説明:

```text
hello from EC2 という文字列を、
ec2_test.txt に保存する
```

内容を確認する。

```bash
cat ec2_test.txt
```

---

## 21. EC2 から S3 へアップロードする

```bash
aws s3 cp ec2_test.txt s3://バケット名/
```

説明:

```text
EC2 内の ec2_test.txt を S3 バケットへコピーする
```

S3 コンソールでも、ファイルが追加されていることを確認する。

---

## 22. 削除する

削除順の例:

```text
1. EC2インスタンスを終了
2. S3バケット内のオブジェクトを削除
3. S3バケットを削除
4. VPC Endpointを削除
5. Security Groupを削除
6. Subnetを削除
7. Route Tableを削除
8. VPCを削除
9. IAM Roleを削除
```

削除する Endpoint:

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
com.amazonaws.ap-northeast-1.s3
```

補足:

```text
Interface Endpointを削除
→ Endpoint用ENIも削除される

S3 Gateway Endpointを削除
→ Route TableのS3向けルートも削除される
```

最後に、不要な EBS Volume が残っていないか確認する。

