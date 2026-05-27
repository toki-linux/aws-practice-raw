# Private Subnetの外部通信失敗確認とSecurity Groupの理解

## 概要

このドキュメントは、AWSのPublic Subnet / Private Subnet構成で、Private EC2からインターネットへ出られないことを確認し、さらにSecurity Groupの通信方向や `My IP` の意味を整理した学習記録です。

今回の実践では、Public EC2からPrivate EC2へSSH接続できる状態を作成したうえで、Private EC2から外部サイトへ通信できないことを確認しました。

また、Public EC2とPrivate EC2間の通信は、同じVPC内であればルート上は可能だが、実際に通信を通すには受け取る側のSecurity Groupで許可が必要であることを整理しました。

---

## 今回の目的

今回の目的は以下です。

- Private SubnetのEC2からインターネットへ出られないことを確認する
- Private SubnetのルートテーブルにInternet Gatewayへのルートがない意味を理解する
- Public EC2とPrivate EC2の間は、同じVPC内ならルート上は通信できることを理解する
- ただし、Security Groupで許可されていなければ通信できないことを理解する
- `SSH / TCP 22 / My IP` の `My IP` が何を指すのか理解する

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

接続の流れは以下です。

```text
自分のPC
↓ SSH
public-ec2
↓ SSH
private-ec2
```

---

## ルートテーブル構成

### Public Subnet用ルートテーブル

Public Subnetには、Internet Gatewayへのルートを持つルートテーブルを関連付けました。

```text
Destination: 10.0.0.0/16
Target: local

Destination: 0.0.0.0/0
Target: Internet Gateway
```

これにより、Public SubnetにあるEC2はインターネットへ通信できます。

---

### Private Subnet用ルートテーブル

Private Subnetには、Internet Gatewayへのルートを持たないルートテーブルを関連付けました。

```text
Destination: 10.0.0.0/16
Target: local
```

Private Subnet用ルートテーブルには、以下を設定していません。

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

そのため、Private EC2はインターネットへ直接出られません。

---

## Private EC2から外部通信を試した

Private EC2へSSH接続したあと、外部サイトへ通信できるか確認しました。

```bash
curl -I https://aws.amazon.com
```

結果として、通信は失敗しました。

これは、Private Subnetのルートテーブルに外向きのルートがないためです。

```text
private-ec2
↓
外部サイトへ通信したい
↓
Route Tableを見る
↓
0.0.0.0/0 のルートがない
↓
インターネットへ出られない
```

---

## 確認できたこと

今回確認できたことは以下です。

```text
Private SubnetのEC2は、Public IPを持たない
Private SubnetのRoute TableにはIGWへのルートがない
そのため、Private EC2からインターネットへ直接出られない
```

つまり、Private Subnetは以下のような状態です。

```text
外から直接入れない
外へも直接出られない
VPC内通信はできる
```

---

## Private Subnetは完全に孤立しているわけではない

Private EC2はインターネットへ出られません。

しかし、同じVPC内のリソースとは通信経路があります。

理由は、VPC内のルートテーブルには以下のlocalルートがあるためです。

```text
10.0.0.0/16 → local
```

このルートにより、同じVPC内のIPアドレス同士は通信できます。

例:

```text
public-ec2:  10.0.1.x
private-ec2: 10.0.2.x
```

この2つは同じVPC内のIPなので、ルート上は通信可能です。

---

## ただしSecurity Groupで許可が必要

同じVPC内で通信経路があっても、Security Groupで許可されていなければ通信できません。

整理すると以下です。

```text
Route Table
→ そもそも通信経路があるか

Security Group
→ その通信をEC2の入口で許可するか
```

つまり、ルートテーブルだけで通信が通るわけではありません。

---

## 現在のSecurity Group設定

### public-ec2-sg

```text
Inbound:
SSH / TCP 22 / Source: My IP
```

これは、自分のPCからpublic-ec2へのSSHを許可する設定です。

```text
自分のPC
↓ SSH
public-ec2
```

---

### private-ec2-sg

```text
Inbound:
SSH / TCP 22 / Source: public-ec2-sg
```

これは、public-ec2-sgが付いているEC2からprivate-ec2へのSSHを許可する設定です。

```text
public-ec2
↓ SSH
private-ec2
```

---

## public-ec2からprivate-ec2へSSHできる理由

以下の通信を行う場合、

```text
public-ec2
↓ SSH / TCP 22
private-ec2
```

通信を受け取る側は `private-ec2` です。

そのため、確認するSecurity Groupは `private-ec2-sg` です。

`private-ec2-sg` には以下のルールがあります。

```text
SSH / TCP 22 / Source: public-ec2-sg
```

そのため、public-ec2からprivate-ec2へのSSHは許可されます。

---

## private-ec2からpublic-ec2へSSHできるのか

逆方向に、private-ec2からpublic-ec2へSSHしたい場合を考えます。

```text
private-ec2
↓ SSH / TCP 22
public-ec2
```

この場合、通信を受け取る側は `public-ec2` です。

そのため、確認するSecurity Groupは `public-ec2-sg` です。

現在の `public-ec2-sg` が以下のみの場合、

```text
SSH / TCP 22 / Source: My IP
```

private-ec2からpublic-ec2へのSSHは許可されません。

理由は、`My IP` は自分のPCのグローバルIPであり、private-ec2ではないからです。

---

## private-ec2からpublic-ec2へSSHしたい場合

private-ec2からpublic-ec2へSSHしたい場合は、`public-ec2-sg` に以下のようなInbound ruleを追加する必要があります。

```text
public-ec2-sg

Inbound:
SSH / TCP 22 / Source: private-ec2-sg
```

これにより、private-ec2-sgが付いているEC2からpublic-ec2へのSSHを許可できます。

接続するときは、public-ec2のPrivate IPを使います。

```bash
ssh -i key.pem ubuntu@public-ec2のPrivateIP
```

ただし、SSH認証には鍵も必要です。

Security Groupで通信を許可しても、秘密鍵がなければログインできません。

```text
通信許可
+
SSH認証
```

の両方が必要です。

---

## 戻り通信と新規通信の違い

public-ec2からprivate-ec2へSSHしているとき、private-ec2からpublic-ec2へ返事は返っています。

これは、すでに確立したSSH接続の戻り通信です。

Security Groupはステートフルなので、許可した通信の戻り通信は自動で許可されます。

しかし、以下は別です。

```text
private-ec2
↓ 新しくSSH接続を開始
public-ec2
```

これは新規通信です。

そのため、受け取る側である `public-ec2-sg` のInbound ruleで許可が必要です。

---

## My IPとは何か

Security Groupで出てくる `My IP` は、AWSコンソールを操作している自分のPC側のグローバルIPです。

```text
SSH / TCP 22 / Source: My IP
```

これは、以下の意味です。

```text
自分のPCのグローバルIPからだけ、
このEC2へのSSHを許可する
```

---

## My IPはpublic-ec2のPublic IPではない

`My IP` は、public-ec2のPublic IPではありません。

整理すると以下です。

```text
自分のPCのグローバルIP
→ Security GroupのSourceに使う

public-ec2のPublic IP
→ SSHコマンドの接続先に使う
```

例えば、自分のPCからpublic-ec2へSSHする場合は以下です。

```bash
ssh -i key.pem ubuntu@public-ec2のPublicIP
```

このとき、

```text
送信元: 自分のPCのグローバルIP
宛先: public-ec2のPublic IP
```

となります。

そのため、public-ec2-sgのInbound ruleでは、Sourceに `My IP` を指定します。

```text
public-ec2-sg

Inbound:
SSH / TCP 22 / Source: My IP
```

---

## My IPのイメージ

```text
自分のPC
  送信元IP: My IP
        ↓ SSH
public-ec2
  宛先IP: public-ec2のPublic IP
```

つまり、`My IP` は「接続してくる側」のIPです。

public-ec2のPublic IPは「接続される側」のIPです。

---

## 今回の学び

今回の実践で、以下を確認しました。

```text
Private EC2からインターネットへ通信できなかった
原因はPrivate SubnetのRoute Tableに0.0.0.0/0のルートがないこと
Private Subnetは外から入れないだけでなく、そのままだと外へも出られない
同じVPC内であればlocalルートにより通信経路はある
ただし、Security Groupで許可されていなければ通信は通らない
通信を見るときは、受け取る側のSecurity Groupを見る
My IPは自分のPCのグローバルIPであり、EC2のPublic IPではない
```

---

## 通信確認時の考え方

通信できないときは、以下の順番で確認します。

```text
1. どこからどこへ通信しているか
2. 何の通信か
3. ルートテーブル上、通信経路はあるか
4. 受け取る側のSecurity Groupで許可されているか
5. SSHの場合、正しい秘密鍵を使っているか
```

例:

```text
private-ec2 → Internet
→ Route Tableに0.0.0.0/0がないため失敗

public-ec2 → private-ec2 SSH
→ private-ec2-sgでSSHを許可しているため成功

private-ec2 → public-ec2 SSH
→ public-ec2-sgでprivate-ec2-sgからのSSHを許可していなければ失敗
```

---

## 30秒説明

Private Subnetに配置したEC2から、外部サイトへ通信できるか確認しました。

Private Subnetのルートテーブルには `10.0.0.0/16 → local` のみがあり、`0.0.0.0/0 → Internet Gateway` がありません。

そのため、Private EC2からインターネットへ直接出ることはできませんでした。

一方で、同じVPC内であればlocalルートによりPublic EC2とPrivate EC2の間には通信経路があります。

ただし、実際に通信できるかどうかは、受け取る側のSecurity Groupで許可されているかによって決まります。

また、Security Groupの `My IP` は自分のPCのグローバルIPを指し、EC2のPublic IPではないことも確認しました。
