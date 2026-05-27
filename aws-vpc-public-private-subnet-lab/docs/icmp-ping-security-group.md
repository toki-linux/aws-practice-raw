# ICMP / ping / Security Group 検証

## 概要

このドキュメントは、AWSのPublic EC2とPrivate EC2間で、SSHは通るのにpingが通らなかった理由を検証した記録です。

Public Subnetに配置した `public-ec2` と、Private Subnetに配置した `private-ec2` を使い、Security Groupで許可する通信の種類によって、SSHとpingの結果が変わることを確認しました。

---

## 今回の目的

今回の目的は以下です。

- SSHとpingは別の通信であることを理解する
- SSHはTCP 22番を使うことを確認する
- pingはICMPを使うことを確認する
- Security Groupでは通信の種類ごとに許可が必要だと理解する
- 通信方向によって、確認すべきSecurity Groupが変わることを理解する

---

## 前提構成

今回の構成は以下です。

```text
VPC: 10.0.0.0/16

├── Public Subnet: 10.0.1.0/24
│   └── public-ec2
│       ├── Public IPあり
│       ├── Private IP: 10.0.1.x
│       └── public-ec2-sg
│
└── Private Subnet: 10.0.2.0/24
    └── private-ec2
        ├── Public IPなし
        ├── Private IP: 10.0.2.x
        └── private-ec2-sg
```

SSH接続の流れは以下です。

```text
自分のPC
↓ SSH
public-ec2
↓ SSH
private-ec2
```

---

## 既存のSecurity Group設定

### public-ec2-sg

`public-ec2` は、自分のPCからSSHする入口として使います。

そのため、`public-ec2-sg` では、自分のPCからのSSHを許可しました。

```text
public-ec2-sg

Inbound:
SSH / TCP 22 / Source: My IP
```

意味は以下です。

```text
自分のPCからpublic-ec2へのSSHを許可する
```

---

### private-ec2-sg

`private-ec2` は、外部から直接入らず、`public-ec2` からSSHする構成にしました。

そのため、`private-ec2-sg` では、`public-ec2-sg` からのSSHを許可しました。

```text
private-ec2-sg

Inbound:
SSH / TCP 22 / Source: public-ec2-sg
```

意味は以下です。

```text
public-ec2-sgが付いているEC2からprivate-ec2へのSSHを許可する
```

---

## SSH接続の確認

自分のPCから `public-ec2` へSSHしました。

```bash
ssh -i key.pem ubuntu@public-ec2のPublicIP
```

次に、`public-ec2` から `private-ec2` へSSHしました。

```bash
ssh -i /home/ubuntu/private-key.pem ubuntu@private-ec2のPrivateIP
```

このSSH接続は成功しました。

つまり、以下の通信は許可できていました。

```text
public-ec2
↓ SSH / TCP 22
private-ec2
```

このとき確認すべきSecurity Groupは、通信を受け取る側である `private-ec2-sg` です。

```text
受け取る側: private-ec2
確認するSG: private-ec2-sg
必要な許可: SSH / TCP 22 / Source: public-ec2-sg
```

---

## pingを試した

次に、`private-ec2` から `public-ec2` のPrivate IPへpingを実行しました。

```bash
ping public-ec2のPrivateIP
```

例:

```bash
ping 10.0.1.25
```

最初はpingが通りませんでした。

---

## なぜSSHは通るのにpingは通らなかったのか

理由は、SSHとpingは別の通信だからです。

```text
SSH  = TCP 22番
ping = ICMP
```

SSHはTCPの22番ポートを使います。

```text
SSH:
Protocol: TCP
Port: 22
```

一方、pingはICMPを使います。

```text
ping:
Protocol: ICMP
Port: なし
```

そのため、Security GroupでSSHを許可していても、pingが自動的に許可されるわけではありません。

```text
SSH / TCP 22 を許可
≠
ICMP を許可
```

---

## ICMPとは

ICMPは、ネットワークの疎通確認やエラー通知に使われるプロトコルです。

正式名称は以下です。

```text
ICMP = Internet Control Message Protocol
```

代表的な使い方が `ping` です。

pingでは、以下のような通信が行われます。

```text
送信元
↓ ICMP Echo Request
送信先
↓ ICMP Echo Reply
送信元
```

つまりpingは、相手に対して「届いていますか？」と確認し、相手が「届いています」と返す仕組みです。

---

## 今回のping通信の方向

今回試したpingは以下の方向です。

```text
private-ec2
↓ ping / ICMP
public-ec2
```

この場合、通信を受け取る側は `public-ec2` です。

そのため、確認するべきSecurity Groupは `public-ec2-sg` です。

```text
受け取る側: public-ec2
確認するSG: public-ec2-sg
必要な許可: ICMP / Source: private-ec2-sg
```

最初は `public-ec2-sg` にICMPの許可がなかったため、pingが通りませんでした。

---

## ICMPを許可する

AWSコンソールで `public-ec2-sg` のInbound ruleを編集しました。

```text
EC2
→ Security Groups
→ public-ec2-sg
→ Inbound rules
→ Edit inbound rules
→ Add rule
```

追加したルールは以下です。

```text
Type: All ICMP - IPv4
Source: private-ec2-sg
```

意味は以下です。

```text
private-ec2-sgが付いているEC2からpublic-ec2へのICMPを許可する
```

---

## ping再実行

ICMP許可後、`private-ec2` から再度 `public-ec2` のPrivate IPへpingしました。

```bash
ping public-ec2のPrivateIP
```

結果として、pingが通るようになりました。

これにより、最初にpingが通らなかった原因は、ルートではなくSecurity GroupでICMPを許可していなかったためだと確認できました。

---

## 通信方向とSecurity Groupの関係

今回の重要ポイントは、通信方向によって確認するSecurity Groupが変わることです。

### public-ec2 から private-ec2 へSSHする場合

```text
public-ec2
↓ SSH / TCP 22
private-ec2
```

受け取る側は `private-ec2` です。

そのため確認するSecurity Groupは `private-ec2-sg` です。

```text
private-ec2-sg

Inbound:
SSH / TCP 22 / Source: public-ec2-sg
```

---

### private-ec2 から public-ec2 へpingする場合

```text
private-ec2
↓ ping / ICMP
public-ec2
```

受け取る側は `public-ec2` です。

そのため確認するSecurity Groupは `public-ec2-sg` です。

```text
public-ec2-sg

Inbound:
All ICMP - IPv4 / Source: private-ec2-sg
```

---

## SSHとpingの違い

SSHとpingは、どちらも通信確認で使うことがありますが、使っているプロトコルが違います。

```text
SSH:
Protocol: TCP
Port: 22
用途: サーバへログインする

ping:
Protocol: ICMP
Port: なし
用途: ネットワーク疎通確認
```

そのため、SSHできるからといってpingも通るとは限りません。

逆に、pingが通るからといってSSHできるとも限りません。

それぞれに必要な許可が違います。

---

## Security Groupでの考え方

Security Groupでは、通信の種類ごとに許可を設定します。

```text
SSHしたい
→ TCP 22 を許可

HTTP通信したい
→ TCP 80 を許可

HTTPS通信したい
→ TCP 443 を許可

pingしたい
→ ICMP を許可
```

つまり、通信できないときは以下の順番で確認します。

```text
1. どこからどこへ通信しているか
2. 何の通信か
3. 受け取る側のSecurity Groupで許可されているか
```

---

## 今回の確認ポイント

今回の検証では、以下を確認しました。

```text
public-ec2 → private-ec2 にSSH
→ private-ec2-sgで SSH / TCP 22 を許可していたため成功

private-ec2 → public-ec2 にping
→ 最初はpublic-ec2-sgでICMPを許可していなかったため失敗

public-ec2-sgにICMPを許可
→ private-ec2からpublic-ec2へのpingが成功
```

---

## 検証後の対応

ping確認後、追加したICMPルールは不要であれば削除しました。

その後、学習用に作成したリソースも削除しました。

削除の流れは以下です。

```text
1. private-ec2から抜ける
2. public-ec2に戻る
3. public-ec2上の秘密鍵を削除する
4. public-ec2から抜ける
5. EC2 2台をTerminate
6. 自作Security Groupを削除
7. Public Subnet / Private Subnetを削除
8. Public Route Table / Private Route Tableを削除
9. Internet GatewayをDetachして削除
10. VPCを削除
```

---

## 学び

今回の検証で、SSHとpingは別の通信であることを理解しました。

SSHはTCP 22番を使います。

pingはICMPを使います。

そのため、Security GroupでSSHを許可していても、pingは通りません。

また、通信を確認するときは、通信を受け取る側のSecurity Groupを見る必要があります。

```text
public-ec2 → private-ec2
→ 受け取る側は private-ec2
→ private-ec2-sg を確認する

private-ec2 → public-ec2
→ 受け取る側は public-ec2
→ public-ec2-sg を確認する
```

この経験から、通信できない原因を調べるときは、以下の3点を確認することが重要だとわかりました。

```text
1. どこからどこへ
2. 何の通信か
3. 受け取る側のSecurity Groupで許可されているか
```

---

## 30秒説明

Public EC2とPrivate EC2の間で、SSHは通るのにpingが通らない理由を検証しました。

SSHはTCP 22番を使う通信で、pingはICMPを使う通信です。

そのため、Security GroupでSSHを許可していても、pingは別途ICMPを許可しないと通りません。

今回、private-ec2からpublic-ec2へpingしたところ最初は通りませんでした。

通信を受け取る側であるpublic-ec2のSecurity Groupに、private-ec2-sgからのICMPを許可したところ、pingが通るようになりました。

この検証を通して、通信方向、通信の種類、受け取る側のSecurity Groupを見ることの重要性を理解しました。
