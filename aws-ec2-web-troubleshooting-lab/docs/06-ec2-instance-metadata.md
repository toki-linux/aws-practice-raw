# EC2 Instance Metadata / IMDSv2の確認

## 概要

EC2 Instance Metadataを使用し、EC2インスタンス内から自身の情報を取得しました。

AWSコンソールを開いて確認するだけでなく、EC2内部からコマンドでインスタンスID、Availability Zone、インスタンスタイプ、IPアドレスなどを確認できることを学びました。

---

## 目的

この課題の目的は、EC2インスタンス自身がAWS上の自分の情報を取得できる仕組みを理解することです。

Instance Metadataを使うと、EC2内から以下のような情報を取得できます。

```text
インスタンスID
Availability Zone
インスタンスタイプ
プライベートIP
パブリックIP
```

---

## Instance Metadataとは

Instance Metadataは、EC2インスタンス自身が自分自身の情報を取得できる仕組みです。

EC2内から以下の特別なアドレスへアクセスすることで、AWSが持っているそのインスタンス自身の情報を確認できます。

```text
169.254.169.254
```

イメージは以下です。

```text
EC2インスタンス
↓
169.254.169.254 に問い合わせる
↓
AWSが持っている「このEC2自身の情報」を取得する
```

---

## IMDSv2について

今回使用したのはIMDSv2です。

IMDSv2では、メタデータを取得する前に、まずトークンを取得します。

```text
1. トークンを取得する
2. 取得したトークンを付けてメタデータを取得する
```

---

## 実行したコマンド

まず、EC2インスタンス内でトークンを取得しました。

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

トークンが取得できているか確認しました。

```bash
echo "$TOKEN"
```

長い文字列が表示されれば、トークン取得は成功です。

---

## インスタンスIDの取得

```bash
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-id
```

取得できる情報の例：

```text
i-xxxxxxxxxxxxxxxxx
```

---

## Availability Zoneの取得

```bash
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/placement/availability-zone
```

取得できる情報の例：

```text
ap-northeast-1a
```

---

## インスタンスタイプの取得

```bash
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-type
```

取得できる情報の例：

```text
t2.micro
```

---

## プライベートIPの取得

```bash
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/local-ipv4
```

取得できる情報の例：

```text
172.31.x.x
```

---

## パブリックIPの取得

```bash
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4
```

取得できる情報の例：

```text
3.xxx.xxx.xxx
```

パブリックIPが付与されていないインスタンスの場合、値が返らないことがあります。

---

## 一括確認用コマンド

複数のメタデータを一度に確認する場合は、以下のように実行しました。

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

echo "Instance ID:"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-id
echo

echo "Availability Zone:"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/placement/availability-zone
echo

echo "Instance Type:"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/instance-type
echo

echo "Private IPv4:"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/local-ipv4
echo

echo "Public IPv4:"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/public-ipv4
echo
```

---

## 確認できたこと

EC2インスタンス内から、以下の情報を取得できました。

```text
インスタンスID
Availability Zone
インスタンスタイプ
プライベートIP
パブリックIP
```

これにより、AWSコンソールを開かなくても、EC2内部から自分自身の情報を確認できることが分かりました。

---

## Webページへ表示しなかった理由

取得したメタデータをWebページに表示することも可能ですが、今回は行いませんでした。

理由は、インスタンスIDやAZなどはパスワードや秘密鍵ほどの機密情報ではないものの、外部に公開する必要のない内部情報だからです。

また、今回の目的は「取得した情報をHTMLに流すこと」ではなく、「EC2自身の情報をInstance Metadataから取得できることを理解すること」でした。

そのため、Webページには表示せず、ターミナル上での確認にとどめました。

---

## 学び

Instance Metadataは、EC2自身の情報をコマンドで確認できる仕組みです。

今回の理解は以下です。

```text
Instance Metadata
= EC2インスタンス自身が、自分自身の情報を取得できる仕組み
```

初学者の段階では、AWSコンソールで確認する方が直感的ですが、Instance Metadataを使うことで、CLIからEC2自身の情報を確認できるようになります。

これは、複数のEC2を扱う場合や、スクリプトで自動処理を行う場合に役立つと理解しました。

---

## 今回の到達点

今回の課題では、以下を確認できました。

```text
IMDSv2のトークンを取得できた
トークンを使ってメタデータにアクセスできた
インスタンスIDを取得できた
Availability Zoneを取得できた
インスタンスタイプを取得できた
IPアドレスを取得できた
169.254.169.254 がEC2内から使う特別なアドレスだと理解した
```

---

## 補足：公開しない方がよい理由

Instance Metadataで取得できる情報は、すべてを公開Webページに載せる必要はありません。

特に以下のような情報は、公開する必要がない場合は外部に出さない方が安全です。

```text
インスタンスID
IPアドレス
Availability Zone
IAMロール情報
```

今回の学習では、取得できることを確認することを目的とし、取得した値そのものは公開しない方針にしました。
