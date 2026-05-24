# EC2 nginx logs to CloudWatch Logs

## 概要

このリポジトリは、EC2上で動作するnginxのログをCloudWatch Logsへ送信する学習記録です。

EC2にCloudWatch Agentをインストールし、nginxの `access.log` と `error.log` をCloudWatch Logsへ送信しました。

CloudWatch Logs上では、ロググループとログストリームを確認し、EC2内のnginxログがAWSコンソールから確認できることを確認しました。

---

## 目的

この課題の目的は、EC2内のログをCloudWatch Logsで確認する仕組みを理解することです。

- EC2にnginxをインストールする
- nginxのログファイルの場所を確認する
- CloudWatch AgentをEC2にインストールする
- CloudWatch Agentの設定ファイルを作成する
- IAM Roleを使ってCloudWatch Logsへログを送信する
- CloudWatch Logsでロググループ・ログストリームを確認する
- CloudWatch Logsの基本的な仕組みを理解する

---

## 構成

```text
AWS
├── EC2
│   ├── Ubuntu
│   ├── nginx
│   │   ├── /var/log/nginx/access.log
│   │   └── /var/log/nginx/error.log
│   └── CloudWatch Agent
│
├── IAM Role
│   └── CloudWatchAgentServerPolicy
│
└── CloudWatch Logs
    ├── /ec2/nginx/access
    │   └── ログストリーム
    └── /ec2/nginx/error
        └── ログストリーム
```

---

## 使用したAWSサービス

- EC2
- IAM Role
- CloudWatch Logs
- Security Group

---

## 使用した主なLinux / AWS関連コマンド

- `apt`
- `wget`
- `dpkg`
- `systemctl`
- `curl`
- `tail`
- `cat`
- `amazon-cloudwatch-agent-ctl`

詳細は以下にまとめています。

```text
[docs/commands.md](docs/commands.md)
```

---

## 実施内容

### 1. IAM Roleの作成

EC2上のCloudWatch AgentがCloudWatch Logsへログを送信できるように、IAM Roleを作成しました。

Role作成時の設定は以下です。

```text
信頼されたエンティティタイプ：AWSのサービス
ユースケース：EC2
許可ポリシー：CloudWatchAgentServerPolicy
Role名：ec2-cloudwatch-agent-role
```

このRoleをEC2インスタンスに付与しました。

---

### 2. EC2インスタンスの作成

UbuntuのEC2インスタンスを作成しました。

Security Groupでは、SSH接続用に22番ポートを許可しました。

```text
SSH: 22
```

必要に応じて、ブラウザからnginxの表示確認をするためにHTTPも許可しました。

```text
HTTP: 80
```

---

### 3. nginxのインストール

EC2へSSH接続し、nginxをインストールしました。

```bash
sudo apt update
sudo apt install -y nginx
```

nginxの状態を確認しました。

```bash
systemctl status nginx
```

EC2内部からnginxにアクセスできることを確認しました。

```bash
curl localhost
```

---

### 4. nginxログの確認

nginxのログファイルは以下にあります。

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

ログファイルが存在するか確認しました。

```bash
ls -l /var/log/nginx/
```

---

### 5. CloudWatch Agentのインストール

CloudWatch Agentのインストール用ファイルをダウンロードしました。

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

ダウンロードした `.deb` ファイルを使って、CloudWatch Agentをインストールしました。

```bash
sudo dpkg -i ./amazon-cloudwatch-agent.deb
```

インストール確認を行いました。

```bash
dpkg -l | grep amazon-cloudwatch-agent
```

---

### 6. CloudWatch Agentの設定ファイル作成

CloudWatch Agentに「どのログファイルをCloudWatch Logsへ送るか」を指定するため、設定ファイルを作成しました。

設定ファイルのパスは以下です。

```text
/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

作成コマンドです。

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

今回の設定では、以下の2つのログを送信対象にしました。

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

設定ファイルの内容は以下に保存しています。

```text
configs/cloudwatch-agent-config.json
```

---

### 7. CloudWatch Agentの起動

設定ファイルをCloudWatch Agentに読み込ませて起動しました。

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

CloudWatch Agentの状態を確認しました。

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

`running` と表示されれば起動成功です。

---

### 8. nginxログの発生

CloudWatch Logsへ送信するログを発生させるため、EC2内部からnginxへアクセスしました。

```bash
curl localhost
curl localhost/notfound
```

EC2内のnginxログも確認しました。

```bash
sudo tail -n 20 /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
```

---

### 9. CloudWatch Logsで確認

AWSコンソールでCloudWatchを開き、以下を確認しました。

```text
CloudWatch
→ ログ
→ ロググループ
```

今回作成されたロググループは以下です。

```text
/ec2/nginx/access
/ec2/nginx/error
```

ロググループを開き、ログストリーム内でnginxのアクセスログが表示されることを確認しました。

---

## CloudWatch Logsの理解

今回の実践で、CloudWatch LogsはEC2に直接アタッチするものではないと理解しました。

流れは以下です。

```text
EC2内のnginxログ
↓
CloudWatch Agentが読み取る
↓
IAM Roleの権限でCloudWatch Logsへ送信
↓
CloudWatch Logsに保存
↓
AWSコンソールで確認
```

---

## 用語整理

### CloudWatch Agent

EC2内のログやメトリクスをCloudWatchへ送信するためのソフトウェアです。

今回の場合、nginxのログファイルを読み取り、CloudWatch Logsへ送信する役割を持っています。

---

### IAM Role

EC2にAWS操作権限を渡す仕組みです。

今回の場合、EC2上のCloudWatch AgentがCloudWatch Logsへログを送信するために使用しました。

---

### CloudWatchAgentServerPolicy

CloudWatch AgentがCloudWatchへログやメトリクスを送信するために必要なAWS管理ポリシーです。

---

### ロググループ

CloudWatch Logsでログをまとめるための箱です。

今回作成されたロググループは以下です。

```text
/ec2/nginx/access
/ec2/nginx/error
```

---

### ログストリーム

ロググループの中にあるログの流れです。

今回の設定では、ログストリーム名にEC2のインスタンスIDを使いました。

```json
"log_stream_name": "{instance_id}"
```

---

### ログイベント

CloudWatch Logsに保存された実際のログ1行1行です。

例：

```text
GET / HTTP/1.1" 200
GET /notfound HTTP/1.1" 404
```

---

## 発生したトラブル

### CloudWatch Agentのダウンロードで403 Forbidden

最初、CloudWatch AgentをダウンロードするときにURLを誤り、以下のようなエラーが出ました。

```text
ERROR 403: Forbidden.
```

これはSecurity GroupでHTTPSを許可していないことが原因ではなく、ダウンロードURLが誤っていたことが原因でした。

正しいURLは以下です。

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

詳細は以下にまとめています。

```text
docs/troubleshooting.md
```

---

## 学び

- CloudWatch LogsはEC2にアタッチして使うものではない
- EC2内にCloudWatch Agentをインストールして使う
- CloudWatch Agentは設定ファイルに従ってログを送信する
- IAM RoleはEC2がCloudWatch Logsへログを送るための権限として使う
- ロググループはログを入れる箱
- ログストリームはロググループ内のログの流れ
- 404はaccess.logには出るが、error.logに必ず出るとは限らない
- コマンドは長くても、役割ごとに分けると理解しやすい

---

## 削除チェック

検証後、課金防止と整理のため、以下を削除・確認しました。

- EC2インスタンスの終了
- CloudWatch Logsのロググループ削除
  - `/ec2/nginx/access`
  - `/ec2/nginx/error`
- 不要なSecurity Groupの削除
- IAM Roleを削除するか確認
- Key Pairを削除するか確認
- EBSボリュームが残っていないか確認

---

## まとめ

今回の実践では、EC2上のnginxログをCloudWatch Agentで収集し、CloudWatch Logsへ送信しました。

AWSコンソール上でロググループとログストリームを確認し、EC2内のログがCloudWatch Logsへ送信されていることを確認できました。

特に、CloudWatch LogsはEC2の中を直接見ているのではなく、EC2内のCloudWatch AgentがIAM Roleの権限を使ってログを送信している、という仕組みを理解できました。
