# CloudWatch Logs実践 トラブルシューティング

## 概要

このファイルでは、EC2上のnginxログをCloudWatch Logsへ送信する実践で発生したトラブルや、確認ポイントをまとめます。

---

## 1. CloudWatch Agentのダウンロードで403 Forbidden

### 症状

CloudWatch Agentをダウンロードしようとしたところ、以下のエラーが出ました。

```text
ERROR 403: Forbidden.
```

実行したコマンド例：

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/amazon-cloudwatch-agent.deb
```

---

### 原因

URLが誤っていました。

この場合、403 ForbiddenはSecurity GroupでHTTPSが許可されていないことが原因ではありません。

実際には、EC2からS3への接続自体はできていました。

```text
Connecting to s3.amazonaws.com... connected.
HTTP request sent, awaiting response... 403 Forbidden
```

これは、通信できているが、そのURLのオブジェクトへアクセスできない状態です。

---

### 対応

正しいURLでダウンロードしました。

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
```

---

### 学び

EC2から外部のHTTPSへアクセスする場合、基本的にはSecurity Groupのインバウンドではなくアウトバウンドが関係します。

通常、Security Groupのアウトバウンドは全許可になっているため、今回の原因はポートではなくURLでした。

---

## 2. CloudWatch Agentをインストールしたが起動していない

### 状態

CloudWatch Agentはインストールできているが、まだ起動していない状態がありました。

確認コマンド：

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

---

### 原因

CloudWatch Agentは、インストールしただけではログ送信を開始しません。

どのログファイルを送るかを設定した `config.json` を作成し、その設定を読み込ませて起動する必要があります。

---

### 対応

設定ファイルを作成しました。

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

その後、設定ファイルを読み込ませて起動しました。

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

---

## 3. 設定ファイルに書き込めない

### 原因

`/opt/aws/amazon-cloudwatch-agent/bin/` 配下は通常ユーザーでは書き込めません。

管理者権限が必要です。

---

### 対応

`sudo` を付けて編集します。

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

または、`tee` を使って作成する方法もあります。

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/bin/config.json > /dev/null <<'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/nginx/access.log",
            "log_group_name": "/ec2/nginx/access",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
          },
          {
            "file_path": "/var/log/nginx/error.log",
            "log_group_name": "/ec2/nginx/error",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
EOF
```

---

## 4. CloudWatch Logsにロググループが出ない

### 確認ポイント

まず、CloudWatch Agentが起動しているか確認します。

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

`running` と表示されているか確認します。

---

### Agent自身のログを確認

```bash
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

ここに設定ミスや権限エラーが出ている可能性があります。

---

### IAM Roleを確認

EC2にCloudWatch Logsへ送信する権限があるRoleが付いているか確認します。

今回必要なポリシー：

```text
CloudWatchAgentServerPolicy
```

---

### nginxログが存在するか確認

```bash
ls -l /var/log/nginx/
```

以下が存在するか確認します。

```text
access.log
error.log
```

---

### nginxログを発生させる

CloudWatch Logsに送るログがないと、表示されないことがあります。

```bash
curl localhost
curl localhost/notfound
```

その後、EC2内でログを確認します。

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

---

## 5. ロググループはあるがログストリームが見つからない

### 原因

AWSコンソールの画面表示によって、ログストリームの表示場所が分かりづらい場合があります。

---

### 確認する場所

```text
CloudWatch
→ ログ
→ ロググループ
→ /ec2/nginx/access
```

その中で以下を探します。

```text
ログストリーム
Log streams
ストリーム
ログイベント
```

今回の設定では、ログストリーム名はEC2のインスタンスIDになります。

```json
"log_stream_name": "{instance_id}"
```

---

## 6. error.logにログが出ない

### 理由

`curl localhost/notfound` で404を発生させても、必ず `error.log` に出るとは限りません。

404は基本的に `access.log` には記録されます。

---

### 確認

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

以下のようなログがあれば、アクセスログは記録されています。

```text
GET /notfound HTTP/1.1" 404
```

---

## 7. IAM Roleを付け忘れた

### 症状

CloudWatch Agentは起動していても、CloudWatch Logsへ送れないことがあります。

Agentログに権限エラーが出る可能性があります。

---

### 対応

EC2にIAM Roleを付与します。

```text
EC2
→ 対象インスタンス
→ アクション
→ セキュリティ
→ IAMロールを変更
```

付与するRoleには以下のポリシーを付けます。

```text
CloudWatchAgentServerPolicy
```

---

## 8. 削除時に忘れやすいもの

CloudWatch Logsのロググループは、EC2を削除しても自動では消えません。

今回作成されたロググループ：

```text
/ec2/nginx/access
/ec2/nginx/error
```

検証後は必要に応じて削除します。

```text
CloudWatch
→ ログ
→ ロググループ
→ 対象ロググループを選択
→ 削除
```

---

## まとめ

CloudWatch Logsでログが見えない場合は、以下の順番で確認します。

```text
1. nginxログがEC2内に存在するか
2. CloudWatch Agentがインストールされているか
3. config.jsonが正しく書けているか
4. Agentがrunningになっているか
5. EC2にIAM Roleが付いているか
6. CloudWatchAgentServerPolicyが付与されているか
7. Agent自身のログにエラーが出ていないか
8. CloudWatch Logsのロググループ・ログストリームを確認したか
```
