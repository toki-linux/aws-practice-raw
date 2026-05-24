# CloudWatch Logs実践 使用コマンド集

## 概要

このファイルでは、EC2上のnginxログをCloudWatch Logsへ送信するために使用したコマンドを整理します。

今回の流れは以下です。

```text
nginxをインストール
↓
CloudWatch Agentをインストール
↓
設定ファイルを作成
↓
CloudWatch Agentを起動
↓
nginxログを発生させる
↓
CloudWatch Logsで確認
```

---

## 1. nginxをインストールする

### パッケージ一覧を更新

```bash
sudo apt update
```

意味：

```text
Ubuntuが参照するパッケージ一覧を最新化する。
ソフトウェアをインストールする前に実行する。
```

---

### nginxをインストール

```bash
sudo apt install -y nginx
```

意味：

```text
nginxをインストールする。
-y は確認メッセージに自動で yes と答えるオプション。
```

---

### nginxの状態確認

```bash
systemctl status nginx
```

意味：

```text
nginxサービスが起動しているか確認する。
active (running) と表示されれば起動中。
```

---

### EC2内部からnginxにアクセス

```bash
curl localhost
```

意味：

```text
EC2自身からnginxにアクセスする。
nginxが正常に応答するか確認できる。
```

---

## 2. nginxログを確認する

### nginxログディレクトリを確認

```bash
ls -l /var/log/nginx/
```

意味：

```text
nginxのログファイルが存在するか確認する。
```

確認するファイル：

```text
access.log
error.log
```

---

### access.logを確認

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

意味：

```text
nginxのアクセスログを最後の20行だけ表示する。
```

---

### error.logを確認

```bash
sudo tail -n 20 /var/log/nginx/error.log
```

意味：

```text
nginxのエラーログを最後の20行だけ表示する。
```

補足：

```text
404 Not Found は access.log には出るが、error.log に必ず出るとは限らない。
```

---

## 3. CloudWatch Agentをダウンロードする

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

意味：

```text
CloudWatch Agentのインストール用ファイルをEC2にダウンロードする。
```

ポイント：

```text
この時点ではまだインストールされていない。
インストール用ファイルを取得しただけ。
```

---

## 4. CloudWatch Agentをインストールする

```bash
sudo dpkg -i ./amazon-cloudwatch-agent.deb
```

意味：

```text
ダウンロードした .deb ファイルを使って、CloudWatch Agentをインストールする。
```

分解：

```text
sudo
→ 管理者権限で実行する

dpkg
→ Ubuntu/Debian系のパッケージを扱うコマンド

-i
→ install の意味

./amazon-cloudwatch-agent.deb
→ カレントディレクトリにあるインストールファイル
```

---

## 5. CloudWatch Agentのインストール確認

```bash
dpkg -l | grep amazon-cloudwatch-agent
```

意味：

```text
インストール済みパッケージの中から、amazon-cloudwatch-agentを探す。
```

表示例：

```text
ii  amazon-cloudwatch-agent ...
```

`ii` と表示されていれば、インストール済みです。

---

## 6. CloudWatch Agentの設定ファイルを作成する

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

意味：

```text
CloudWatch Agentの設定ファイルを作成・編集する。
```

この設定ファイルでは、以下を指定します。

```text
どのログファイルを送るか
どのロググループへ送るか
ログストリーム名をどうするか
```

---

## 7. 設定ファイルの中身を確認する

```bash
cat /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

意味：

```text
作成した設定ファイルの内容を表示し、正しく書けているか確認する。
```

---

## 8. CloudWatch Agentに設定ファイルを読み込ませて起動する

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

意味：

```text
config.jsonをCloudWatch Agentに読み込ませて、EC2モードで起動する。
```

分解：

```text
sudo
→ 管理者権限で実行する

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl
→ CloudWatch Agentを操作する専用コマンド

-a fetch-config
→ 設定ファイルを読み込む

-m ec2
→ EC2上で動かすモード

-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
→ この設定ファイルを使う

-s
→ 設定を読み込んだあと起動する
```

覚え方：

```text
CloudWatch Agentにconfig.jsonを読ませて起動するコマンド
```

---

## 9. CloudWatch Agentの状態を確認する

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

意味：

```text
CloudWatch Agentが起動しているか確認する。
```

確認ポイント：

```text
running
```

`running` と表示されていれば、Agentは起動しています。

---

## 10. CloudWatch Agent自身のログを確認する

```bash
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

意味：

```text
CloudWatch Agent自身の動作ログを確認する。
```

使う場面：

```text
CloudWatch Logsにログが表示されない
Agentが起動しない
設定ファイルにミスがありそう
IAM Roleの権限エラーがありそう
```

---

## 11. nginxログを発生させる

### 正常アクセス

```bash
curl localhost
```

意味：

```text
EC2内部からnginxに正常アクセスする。
access.logに 200 のログが記録される。
```

---

### 存在しないパスにアクセス

```bash
curl localhost/notfound
```

意味：

```text
存在しないURLにアクセスして404を発生させる。
access.logに 404 のログが記録される。
```

---

## 12. CloudWatch Logsで確認する

AWSコンソールで以下を確認します。

```text
CloudWatch
→ ログ
→ ロググループ
```

今回作成されるロググループ：

```text
/ec2/nginx/access
/ec2/nginx/error
```

ロググループを開き、ログストリームを選択すると、実際のログイベントを確認できます。

---

## 今回の最小コマンドセット

```bash
sudo apt update
sudo apt install -y nginx
systemctl status nginx
curl localhost

wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i ./amazon-cloudwatch-agent.deb
dpkg -l | grep amazon-cloudwatch-agent

ls -l /var/log/nginx/

sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
cat /opt/aws/amazon-cloudwatch-agent/bin/config.json

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

curl localhost
curl localhost/notfound

sudo tail -n 20 /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
```

---

## コマンドの役割まとめ

```text
apt
→ nginxをインストールする

wget
→ CloudWatch Agentのインストーラーを取得する

dpkg -i
→ CloudWatch Agentをインストールする

nano config.json
→ どのログを送るか設定する

amazon-cloudwatch-agent-ctl
→ CloudWatch Agentを操作する

curl
→ nginxログを発生させる

tail
→ EC2内のログを確認する
```
