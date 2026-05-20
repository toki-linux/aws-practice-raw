# EC2 User DataによるWebサーバ自動構築

## 概要

EC2インスタンス作成時にUser Dataを設定し、初回起動時にnginxのインストールとWebページ作成を自動実行しました。

これにより、インスタンス作成後にSSH接続して手動で設定するのではなく、起動時に必要な環境を自動で用意できることを確認しました。

---

## 目的

この課題の目的は、EC2のUser Dataを使い、インスタンス初回起動時の初期設定を自動化することです。

これまでの流れでは、EC2作成後に手動で以下を行っていました。

```text
EC2作成
↓
SSH接続
↓
apt update
↓
nginxインストール
↓
index.html作成
↓
nginx起動
```

User Dataを使うことで、この作業をEC2起動時に自動実行できるようになります。

```text
EC2作成時にUser Dataを設定
↓
EC2初回起動
↓
cloud-initがUser Dataを読み込む
↓
root権限でスクリプトを実行
↓
nginxインストール・HTML作成・nginx起動
```

---

## 使用したUser Data

EC2作成画面の「高度な詳細」から、User Dataに以下のスクリプトを設定しました。

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx

cat > /var/www/html/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>EC2 User Data Lab</title>
</head>
<body>
  <h1>Hello from EC2 User Data</h1>
  <p>This web page was created automatically when the EC2 instance launched.</p>
</body>
</html>
EOF

systemctl enable nginx
systemctl start nginx
```

---

## 実行された処理

User Dataによって、初回起動時に以下の処理が自動で行われました。

```text
1. パッケージ情報の更新
2. nginxのインストール
3. /var/www/html/index.html の作成
4. nginxの自動起動設定
5. nginxの起動
```

---

## 確認したこと

インスタンス起動後、ブラウザからパブリックIPへアクセスし、Webページが表示されることを確認しました。

```text
http://パブリックIP
```

EC2内でも以下を確認しました。

```bash
systemctl status nginx
```

```bash
curl http://localhost
```

また、User Dataが実行されたログを確認しました。

```bash
sudo cat /var/log/cloud-init-output.log
```

---

## cloud-initについて

User Dataは、Ubuntu内の`cloud-init`という仕組みによって処理されます。

`cloud-init`は、クラウド上で作成されたLinuxサーバの初回セットアップを行う仕組みです。

```text
EC2初回起動
↓
Ubuntuが起動
↓
cloud-initが動作
↓
User Dataを読み込む
↓
root権限でスクリプトを実行
```

cronのように定期実行するものではなく、基本的には初回起動時の初期設定として使われます。

---

## 権限について

User Dataのスクリプトは、基本的にroot権限で実行されます。

そのため、以下のような操作に`sudo`を付けなくても実行できます。

```bash
apt-get install -y nginx
systemctl start nginx
cat > /var/www/html/index.html
```

通常の一般ユーザーでは権限が足りない操作でも、User Dataでは初期設定として実行できます。

---

## 学び

User Dataを使うことで、EC2インスタンス作成後の手動設定を減らせることを学びました。

特に、同じ構成のサーバを何度も作る場合、毎回SSH接続して手作業で設定するのではなく、起動時に自動で環境を用意できます。

今回の理解は以下です。

```text
User Data
= EC2インスタンス初回起動時に、初期設定を自動実行する仕組み
```

正確には、環境そのものを保存しておくというより、起動時に環境を自動構築するための命令を書いておく機能だと理解しました。

---

## 注意点

User Dataを使うこと自体は無料です。

ただし、User Dataの中で実行する内容によっては、別のAWSサービスや通信量に料金が発生する可能性があります。

今回のようにnginxをインストールしてWebページを作成するだけであれば、主に気にするのは以下です。

```text
EC2の起動時間
EBSボリューム
パブリックIPv4
通信量
```

また、User Dataは基本的に初回起動時のみ実行されます。

インスタンスを停止して再度起動しても、通常は同じUser Dataが毎回再実行されるわけではありません。
