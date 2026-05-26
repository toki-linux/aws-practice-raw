# Troubleshooting Notes

## 概要

このドキュメントは、Public Subnet + Private Subnet 構成の実践中に詰まった点と、その解決方法をまとめたものです。

---

## 1. private-ec2-sgでSecurity GroupをSourceに指定できなかった

### 発生したこと

private-ec2のSecurity Groupで、public-ec2からのSSHだけを許可しようとしました。

設定したかった内容は以下です。

```text
private-ec2-sg

Inbound:
SSH / TCP 22 / Source: public-ec2-sg
```

しかし、以下のようなエラーが出ました。

```text
既存の IPv4 CIDR ルールに 1 つの 参照先のグループ ID を指定することはできません。
```

---

### 原因

既存のIPv4 CIDRルールに対して、Security Group IDを指定しようとしていたことが原因でした。

Sourceには大きく以下の種類があります。

```text
IPv4 CIDR:
203.x.x.x/32
10.0.1.0/24
0.0.0.0/0

Security Group:
sg-xxxxxxxxxxxxxxxxx
```

IPv4 CIDR用のルールに、`sg-xxxx` のようなSecurity Group IDを入れることはできません。

---

### 解決方法

既存のIPv4 CIDRルールを無理に編集するのではなく、新しいSSHルールを追加しました。

```text
private-ec2-sg
→ Inbound rules
→ Edit inbound rules
→ Add rule
```

追加した内容は以下です。

```text
Type: SSH
Protocol: TCP
Port: 22
Source: public-ec2-sg
```

これにより、public-ec2-sgが付いているEC2からのみ、private-ec2へSSH接続できるようになりました。

---

### 学び

Security GroupのSourceには、IPv4 CIDRとSecurity Group参照がある。

```text
IPアドレス範囲で許可する
→ IPv4 CIDR

特定のSecurity Groupを持つEC2から許可する
→ Security Group参照
```

既存ルールの種類を変えようとするとエラーになる場合があるため、新しくルールを追加するとわかりやすい。

---

## 2. public-ec2からprivate-ec2へSSHしたらPermission deniedになった

### 発生したこと

public-ec2からprivate-ec2へSSHしようとしました。

```bash
ssh -i private-key.pem ubuntu@private-ec2のPrivateIP
```

しかし、以下のようなエラーが出ました。

```text
Permission denied
```

---

### 原因

private-ec2用の秘密鍵をpublic-ec2上に置けていなかったことが原因でした。

SSHでは、接続元に秘密鍵があり、接続先に対応する公開鍵が登録されている必要があります。

```text
接続元:
秘密鍵を持っている

接続先:
authorized_keysに対応する公開鍵がある
```

今回の場合、接続元はpublic-ec2です。

そのため、public-ec2上でprivate-ec2用の秘密鍵を使える必要がありました。

---

### 解決方法

自分のPCからpublic-ec2へ、秘密鍵を一時的にコピーしました。

```bash
scp -i key.pem key.pem ubuntu@public-ec2のPublicIP:/home/ubuntu/private-key.pem
```

public-ec2上で権限を変更しました。

```bash
chmod 400 /home/ubuntu/private-key.pem
```

その後、private-ec2へSSH接続しました。

```bash
ssh -i /home/ubuntu/private-key.pem ubuntu@private-ec2のPrivateIP
```

これで接続できました。

---

### 学び

public-ec2からprivate-ec2へSSHする場合、public-ec2側にprivate-ec2用の秘密鍵が必要です。

ただし、秘密鍵を踏み台サーバに置きっぱなしにするのは危険です。

今回は学習用として一時的にコピーし、確認後に削除しました。

```bash
rm /home/ubuntu/private-key.pem
```

---

## 3. known_hostsとauthorized_keysを混同した

### 発生したこと

public-ec2からprivate-ec2へSSHするには、private-ec2の公開鍵をpublic-ec2のknown_hostsに置く必要があるのではないかと考えました。

---

### 実際の整理

known_hostsとauthorized_keysは役割が違います。

```text
known_hosts
→ 接続先サーバが本物か確認するための記録

authorized_keys
→ ログインを許可する公開鍵のリスト
```

SSH認証で重要なのは、接続先サーバ側のauthorized_keysです。

known_hostsは、接続先サーバの身元確認に使われます。

---

### 初回接続時の動き

初めてprivate-ec2へSSHすると、以下のような確認が出ることがあります。

```text
The authenticity of host '10.0.2.x' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

ここで `yes` と入力すると、接続先のホスト情報がknown_hostsに保存されます。

これはログイン認証そのものではありません。

---

### 学び

SSHでは、以下を分けて考える必要があります。

```text
known_hosts
→ 接続先が前回と同じか確認するもの

authorized_keys
→ この秘密鍵を持っている人ならログインしてよいと判断するもの
```

Permission deniedの原因はknown_hostsではなく、鍵認証やユーザー名の問題であることが多い。

---

## 4. Ubuntuのアップデートメッセージが表示された

### 発生したこと

private-ec2へSSH接続したあと、Ubuntuのアップデートに関するメッセージが表示されました。

---

### 原因

これはUbuntuのログイン時に表示される通常のメッセージです。

MOTDと呼ばれるログイン時の案内表示です。

```text
Welcome to Ubuntu
System information
updates can be applied
```

---

### 学び

このメッセージが表示されたということは、private-ec2へのSSHログイン自体は成功しています。

ただし、今回のprivate-ec2はPrivate Subnetにあり、NAT Gatewayも設定していません。

そのため、インターネットへ出る通信は基本的にできません。

```text
private-ec2
↓
Internet
```

この通信には、NAT Gatewayなどの追加構成が必要になります。

---

## 5. private-ec2からpublic-ec2へpingが通らなかった

### 発生したこと

private-ec2からpublic-ec2のPrivate IPへpingを実行しました。

```bash
ping public-ec2のPrivateIP
```

しかし、pingは通りませんでした。

---

### 考えられる原因

pingはSSHとは異なり、ICMPを使用します。

```text
SSH  = TCP 22
ping = ICMP
```

今回、Security GroupではSSHの通信のみ許可していました。

そのため、ICMPが許可されておらず、pingが通らなかった可能性があります。

---

### 解決するなら

public-ec2側のSecurity Groupに、ICMPを許可するInbound ruleを追加します。

例:

```text
public-ec2-sg

Inbound:
Type: All ICMP - IPv4
Source: private-ec2-sg
```

または、検証用に以下のように設定することもできます。

```text
Type: All ICMP - IPv4
Source: 10.0.2.0/24
```

---

### 今回の判断

今回の目的は、public-ec2経由でprivate-ec2へSSH接続することでした。

そのため、pingについては深追いせず、次回以降の検証課題としました。

---

## まとめ

今回のトラブルから、以下を学びました。

- Security GroupのSourceにはIPv4 CIDRとSecurity Group参照がある
- 既存のIPv4 CIDRルールにSecurity Group IDを指定するとエラーになる
- public-ec2からprivate-ec2へSSHするには、public-ec2上でprivate-ec2用の秘密鍵を使える必要がある
- known_hostsとauthorized_keysは役割が違う
- Ubuntuのアップデート表示は通常のログインメッセージ
- pingはSSHとは異なり、ICMP許可が必要
- 今回の目的はSSH接続だったため、ICMPは次回以降の課題にした
