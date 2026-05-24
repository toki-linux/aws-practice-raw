# CloudWatch AgentとCloudWatch Logsの仕組み

## 概要

このファイルでは、EC2内のnginxログがCloudWatch Logsで見えるようになるまでの流れを整理します。

---

## 全体の流れ

```text
EC2
├── nginx
│   ├── /var/log/nginx/access.log
│   └── /var/log/nginx/error.log
│
└── CloudWatch Agent
        ↓
        IAM Roleの権限で送信

CloudWatch Logs
├── /ec2/nginx/access
└── /ec2/nginx/error
```

---

## CloudWatch LogsはEC2にアタッチするものではない

EBSのように、CloudWatch LogsをEC2にアタッチするわけではありません。

EBSの場合は以下のような流れです。

```text
EBSをEC2にアタッチ
↓
lsblkで確認
↓
mountして使う
```

一方、CloudWatch Logsの場合は以下です。

```text
EC2にCloudWatch Agentをインストール
↓
設定ファイルで送るログを指定
↓
IAM Roleで送信権限を付与
↓
CloudWatch AgentがCloudWatch Logsへ送信
```

---

## 登場するもの

### EC2

nginxが動作しているサーバーです。

今回、nginxは以下のログを出力します。

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

---

### nginx

Webサーバーです。

アクセスがあると `access.log` に記録します。

エラー内容によっては `error.log` に記録します。

---

### CloudWatch Agent

EC2内にインストールするソフトウェアです。

EC2内のログファイルを読み取り、CloudWatch Logsへ送信します。

今回の役割は以下です。

```text
/var/log/nginx/access.log を読む
/var/log/nginx/error.log を読む
CloudWatch Logsへ送る
```

---

### config.json

CloudWatch Agentの設定ファイルです。

ここに以下を指定します。

```text
どのログファイルを送るか
どのロググループへ送るか
ログストリーム名をどうするか
```

今回の例です。

```json
{
  "file_path": "/var/log/nginx/access.log",
  "log_group_name": "/ec2/nginx/access",
  "log_stream_name": "{instance_id}"
}
```

意味：

```text
/var/log/nginx/access.log を
/ec2/nginx/access というロググループへ送り、
ログストリーム名にはEC2のインスタンスIDを使う
```

---

### IAM Role

EC2にAWS操作権限を渡す仕組みです。

今回のRoleは、CloudWatch AgentがCloudWatch Logsへログを送信するために使いました。

付与したポリシーは以下です。

```text
CloudWatchAgentServerPolicy
```

---

### CloudWatch Logs

AWS側でログを保管・確認するサービスです。

EC2内のログを直接のぞいているわけではありません。

CloudWatch Agentが送信したログをCloudWatch Logs側で保存し、AWSコンソールから確認できるようにしています。

---

## ロググループとは

ログをまとめる箱です。

今回作成したロググループは以下です。

```text
/ec2/nginx/access
/ec2/nginx/error
```

イメージ：

```text
ロググループ
└── /ec2/nginx/access
```

---

## ログストリームとは

ロググループの中にあるログの流れです。

今回の設定では、ログストリーム名をEC2のインスタンスIDにしました。

```json
"log_stream_name": "{instance_id}"
```

イメージ：

```text
ロググループ
└── /ec2/nginx/access
    └── i-xxxxxxxxxxxxxxxxx
```

---

## ログイベントとは

実際のログ1行1行です。

例：

```text
GET / HTTP/1.1" 200
GET /notfound HTTP/1.1" 404
```

イメージ：

```text
ロググループ
└── ログストリーム
    └── ログイベント
```

---

## メトリクスとログの違い

### メトリクス

状態を数字で表したものです。

例：

```text
CPU使用率 20%
NetworkIn 1MB
StatusCheckFailed 0
```

---

### ログ

起きた出来事の記録です。

例：

```text
GET / HTTP/1.1" 200
GET /notfound HTTP/1.1" 404
permission denied
connect() failed
```

---

## 今回の処理の流れ

```text
1. EC2にnginxをインストールする
2. nginxが /var/log/nginx/ にログを出す
3. EC2にCloudWatch Agentをインストールする
4. config.jsonで送信対象ログを指定する
5. EC2にIAM RoleでCloudWatch Logsへの送信権限を付ける
6. CloudWatch Agentを起動する
7. AgentがログをCloudWatch Logsへ送る
8. AWSコンソールでロググループ・ログストリームを確認する
```

---

## 30秒説明

EC2内のnginxログをCloudWatch Logsで確認するには、EC2にCloudWatch Agentをインストールします。

CloudWatch Agentの設定ファイルで、送信したいログファイルとして `/var/log/nginx/access.log` と `/var/log/nginx/error.log` を指定します。

EC2にはCloudWatch Logsへログを送信するためのIAM Roleを付与します。

その後、CloudWatch Agentを起動すると、指定したログがCloudWatch Logsへ送信され、AWSコンソールからロググループとログストリームとして確認できます。
