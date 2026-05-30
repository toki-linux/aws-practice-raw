# 構築手順

## 1. 構成の前提

今回の EC2 は Private Subnet に配置する。

```text
Public IP: なし
SSH 接続: 使用しない
HTTP 公開: しない
接続方法: Systems Manager Session Manager
```

CloudWatch Logs の仕組みを理解することが目的なので、nginx は使わず、自作ログファイルを送信する。

---

## 2. VPC を作成する

例：

```text
Name: toki-cloudwatch-vpc
IPv4 CIDR: 10.0.0.0/16
```

---

## 3. Private Subnet を作成する

例：

```text
Name: toki-cloudwatch-private-subnet
IPv4 CIDR: 10.0.1.0/24
```

---

## 4. Private Subnet 用 Route Table を作成する

例：

```text
Name: toki-cloudwatch-private-rt
```

Private Subnet と関連付ける。

今回、Internet Gateway と NAT Gateway は作成しない。

基本ルートは次の local ルートのみ。

```text
10.0.0.0/16 → local
```

---

## 5. EC2 用 IAM Role を作成する

EC2 用の IAM Role を作成する。

例：

```text
toki-private-ec2-role
```

以下の AWS 管理ポリシーを付与する。

```text
AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy
```

役割：

```text
AmazonSSMManagedInstanceCore
→ EC2 を Systems Manager で管理するため

CloudWatchAgentServerPolicy
→ CloudWatch Agent が CloudWatch Logs へログを送るため
```

---

## 6. EC2 用 Security Group を作成する

例：

```text
Name: toki-cloudwatch-private-ec2-sg
```

今回は外部公開しないため、Inbound Rule は不要。

```text
Inbound:
なし

Outbound:
許可
```

SSH の TCP 22 番、HTTP の TCP 80 番は開放しない。

---

## 7. VPC Endpoint 用 Security Group を作成する

例：

```text
Name: toki-vpc-endpoint-sg
```

Inbound Rule：

```text
Type: HTTPS
Port: 443
Source: toki-cloudwatch-private-ec2-sg
```

意味：

```text
EC2 から VPC Endpoint へ HTTPS で通信することを許可する
```

---

## 8. SSM 用 VPC Endpoint を作成する

以下の Interface Endpoint を作成する。

```text
com.amazonaws.ap-northeast-1.ssm
com.amazonaws.ap-northeast-1.ssmmessages
com.amazonaws.ap-northeast-1.ec2messages
```

設定例：

```text
VPC: toki-cloudwatch-vpc
Subnet: toki-cloudwatch-private-subnet
Security Group: toki-vpc-endpoint-sg
Private DNS: 有効
```

---

## 9. CloudWatch Logs 用 VPC Endpoint を作成する

以下の Interface Endpoint を作成する。

```text
com.amazonaws.ap-northeast-1.logs
```

設定例：

```text
VPC: toki-cloudwatch-vpc
Subnet: toki-cloudwatch-private-subnet
Security Group: toki-vpc-endpoint-sg
Private DNS: 有効
```

---

## 10. S3 Gateway Endpoint を作成する

以下の Gateway Endpoint を作成する。

```text
com.amazonaws.ap-northeast-1.s3
```

Private Subnet 用 Route Table と関連付ける。

```text
toki-cloudwatch-private-rt
```

今回は、CloudWatch Agent のインストール用パッケージを取得するために使用した。

---

## 11. Private EC2 を起動する

例：

```text
Name: toki-cloudwatch-private-ec2
AMI: Ubuntu
Subnet: toki-cloudwatch-private-subnet
Public IP: 無効
IAM Role: toki-private-ec2-role
Security Group: toki-cloudwatch-private-ec2-sg
```

---

## 12. Session Manager で接続する

EC2 の画面で対象インスタンスを選択する。

```text
接続
↓
Session Manager
↓
接続
```

---

## 13. SSM Agent の起動状態を確認する

EC2 内で実行する。

```bash
systemctl list-units --type=service | grep -i ssm
```

`active` と表示されれば、SSM Agent が動作している。

---

## 14. テスト用ログファイルを作成する

```bash
sudo touch /var/log/toki-test.log
sudo chmod 666 /var/log/toki-test.log
echo "first cloudwatch logs test $(date)" >> /var/log/toki-test.log
tail -n 5 /var/log/toki-test.log
```

今回の `chmod 666` は練習用。

実務では、誰でも書き込める権限は緩いため、用途に応じて適切な権限を設定する。

---

## 15. Run Command で CloudWatch Agent をインストールする

AWS コンソールで以下を開く。

```text
Systems Manager
↓
Run Command
↓
Run command
```

コマンドドキュメント：

```text
AWS-ConfigureAWSPackage
```

パラメータ：

```text
Action: Install
Name: AmazonCloudWatchAgent
Version: latest
```

ターゲット：

```text
今回作成した Private EC2 を手動で選択
```

実行後、ステータスが `Success` になることを確認する。

---

## 16. CloudWatch Agent のインストールを確認する

EC2 内で実行する。

```bash
ls -l /opt/aws/amazon-cloudwatch-agent/bin/
```

ファイルが表示されれば、CloudWatch Agent がインストールされている。

---

## 17. CloudWatch Agent の設定ファイルを作成する

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

以下を記述する。

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/toki-test.log",
            "log_group_name": "/aws/ec2/toki-test-log",
            "log_stream_name": "{instance_id}",
            "timezone": "Local"
          }
        ]
      }
    }
  }
}
```

vi で保存して終了する。

```text
Esc
↓
:wq
↓
Enter
```

---

## 18. CloudWatch Agent を起動する

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

意味：

```text
config.json を読み込む
↓
EC2 用の設定として使用する
↓
CloudWatch Agent を起動する
```

---

## 19. CloudWatch Agent の状態を確認する

```bash
systemctl status amazon-cloudwatch-agent
```

以下の表示を確認する。

```text
active (running)
```

---

## 20. ログを追記する

```bash
echo "cloudwatch logs test after agent start $(date)" | sudo tee -a /var/log/toki-test.log
tail -n 5 /var/log/toki-test.log
```

---

## 21. CloudWatch Logs で確認する

AWS コンソールで以下を開く。

```text
CloudWatch
↓
ログ
↓
ロググループ
↓
/aws/ec2/toki-test-log
```

ログストリームを開く。

```text
EC2 のインスタンス ID
```

EC2 内で追記したログが表示されれば成功。

---

## 22. リソースを削除する

削除順の例：

```text
1. EC2 インスタンスを終了
2. CloudWatch Logs のロググループを削除
3. VPC Endpoint を削除
4. Security Group を削除
5. Route Table を削除
6. Subnet を削除
7. VPC を削除
8. IAM Role を削除
```

EBS Volume が残っていないかも確認する。

