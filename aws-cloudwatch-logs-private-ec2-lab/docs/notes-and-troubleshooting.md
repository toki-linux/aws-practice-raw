# 役割の整理・トラブルシューティング

## 1. 今回の全体像

今回実施したことは以下。

```text
Private EC2 を作成
↓
SSM Session Manager で接続
↓
EC2 内にテストログを作成
↓
Run Command で CloudWatch Agent をインストール
↓
CloudWatch Agent の設定ファイルを作成
↓
CloudWatch Agent を起動
↓
CloudWatch Logs へログを送信
↓
AWS コンソールでログを確認
```

---

# 役割の整理

## 2. SSM Agent とは

SSM Agent は、EC2 内で動作する常駐プログラム。

Systems Manager から送られた命令を受け取って、EC2 内で処理を実行する。

```text
Systems Manager
↓
SSM Agent
↓
EC2 内で処理を実行
```

今回、SSM Agent が動いていたため、以下が可能になった。

```text
Session Manager で接続する
Run Command を実行する
```

確認コマンド：

```bash
systemctl list-units --type=service | grep -i ssm
```

---

## 3. CloudWatch Agent とは

CloudWatch Agent は、EC2 内のログやメトリクスを CloudWatch へ送るための常駐プログラム。

今回送信したログ：

```text
/var/log/toki-test.log
```

流れ：

```text
EC2 内のログファイル
↓
CloudWatch Agent
↓
CloudWatch Logs
```

SSM Agent と CloudWatch Agent は別の役割を持つ。

```text
SSM Agent
→ AWS から EC2 を操作するため

CloudWatch Agent
→ EC2 から CloudWatch へ情報を送るため
```

---

## 4. Agent は複数入れられる

EC2 に Agent は1つしか入れられないわけではない。

目的ごとに複数の Agent を入れられる。

```text
EC2
├── SSM Agent
├── CloudWatch Agent
└── その他の Agent
```

SSM Agent は Ubuntu AMI に最初から入っていた。

CloudWatch Agent は全員が必要とするわけではないため、今回は後から追加した。

---

## 5. Run Command とは

Run Command は、Systems Manager 経由で EC2 に命令を送る機能。

EC2 に直接ログインして手作業する代わりに、AWS コンソールから作業を実行できる。

```text
AWS コンソール
↓
Systems Manager Run Command
↓
SSM Agent
↓
EC2 内で処理を実行
```

今回送った命令：

```text
CloudWatch Agent をインストールして
```

---

## 6. AWS-ConfigureAWSPackage とは

今回 Run Command で選択したコマンドドキュメント。

```text
AWS-ConfigureAWSPackage
```

Systems Manager のコマンドドキュメントは、実行する処理を定義した命令テンプレート。

今回入力したパラメータ：

```text
Action: Install
Name: AmazonCloudWatchAgent
Version: latest
```

意味：

```text
最新版の CloudWatch Agent をインストールする
```

---

## 7. IAM Role と IAM Policy

今回、EC2 用 IAM Role に以下のポリシーを付与した。

```text
AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy
```

### AmazonSSMManagedInstanceCore

```text
EC2 を Systems Manager から管理するための権限
```

これにより、Session Manager や Run Command を使用できる。

### CloudWatchAgentServerPolicy

```text
CloudWatch Agent が CloudWatch Logs へログを送るための権限
```

IAM Role は1つでよい。

```text
EC2
└── IAM Role
    ├── AmazonSSMManagedInstanceCore
    └── CloudWatchAgentServerPolicy
```

---

## 8. VPC Endpoint の役割

今回の Private EC2 は、Public IP なし、NAT Gateway なしの構成。

そのため、AWS サービスへ通信する経路として VPC Endpoint を作成した。

### SSM 用 Endpoint

```text
ssm
ssmmessages
ec2messages
```

役割：

```text
Private EC2
↓
Systems Manager
```

### CloudWatch Logs 用 Endpoint

```text
logs
```

役割：

```text
CloudWatch Agent
↓
CloudWatch Logs
```

### S3 Gateway Endpoint

```text
s3
```

役割：

```text
Private EC2
↓
CloudWatch Agent のパッケージ取得
```

覚え方：

```text
SSM 接続用は3つ
ログ送信用は logs
パッケージ取得用は s3
```

正式名称は必要なときに確認すればよい。

---

## 9. config.json の役割

CloudWatch Agent をインストールしただけでは、どのログを送信するか判断できない。

そこで、設定ファイルを作成した。

```text
/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

設定した内容：

```json
{
  "file_path": "/var/log/toki-test.log",
  "log_group_name": "/aws/ec2/toki-test-log",
  "log_stream_name": "{instance_id}"
}
```

意味：

```text
/var/log/toki-test.log を読み取る
↓
/aws/ec2/toki-test-log へ送る
↓
EC2 のインスタンス ID をログストリーム名にする
```

---

## 10. ロググループとログストリーム

CloudWatch Logs では、ログは次のように整理される。

```text
ロググループ
└── ログストリーム
    └── 実際のログ
```

今回の構成：

```text
/aws/ec2/toki-test-log
└── EC2 のインスタンス ID
    └── cloudwatch logs test after agent start ...
```

---

# Linux コマンドの整理

## 11. SSM Agent を確認する

```bash
systemctl list-units --type=service | grep -i ssm
```

意味：

```text
サービス一覧を表示
↓
ssm を含む行だけ表示
```

---

## 12. 空のログファイルを作る

```bash
sudo touch /var/log/toki-test.log
```

意味：

```text
/var/log/toki-test.log という空ファイルを作成する
```

---

## 13. ファイルにログを追記する

```bash
echo "first cloudwatch logs test $(date)" >> /var/log/toki-test.log
```

意味：

```text
現在日時を含む文字列を作る
↓
ログファイルの末尾へ追加する
```

違い：

```text
>  上書き
>> 追記
```

---

## 14. ログファイルの末尾を確認する

```bash
tail -n 5 /var/log/toki-test.log
```

意味：

```text
ファイルの末尾5行を表示する
```

---

## 15. CloudWatch Agent が入ったか確認する

```bash
ls -l /opt/aws/amazon-cloudwatch-agent/bin/
```

意味：

```text
CloudWatch Agent の実行ファイルが存在するか確認する
```

---

## 16. CloudWatch Agent の状態を確認する

```bash
systemctl status amazon-cloudwatch-agent
```

確認する表示：

```text
active (running)
```

---

## 17. tee でログを追記する

```bash
echo "cloudwatch logs test after agent start $(date)" | sudo tee -a /var/log/toki-test.log
```

意味：

```text
echo で文字列を作る
↓
パイプで tee に渡す
↓
管理者権限でファイル末尾へ追記する
```

`-a` は append の意味。

```text
append
→ 末尾へ追加
```

---

# 詰まった点

## 18. Run Command が最初に失敗した

最初に `AWS-ConfigureAWSPackage` を使用して CloudWatch Agent をインストールしたとき、Run Command がエラーになった。

その時点では、以下の Endpoint は作成済みだった。

```text
ssm
ssmmessages
ec2messages
logs
```

しかし、S3 Gateway Endpoint は未作成だった。

その後、以下を追加した。

```text
com.amazonaws.ap-northeast-1.s3
```

Route Table：

```text
Private Subnet 用 Route Table
```

その後、Run Command を再実行すると成功した。

### 考えられる原因

Private EC2 が CloudWatch Agent のパッケージ取得先へ到達できなかった可能性が高い。

```text
Systems Manager から命令は届いた
↓
しかし、インストール用パッケージを取得できなかった
↓
S3 Gateway Endpoint を追加
↓
パッケージ取得が可能になった
↓
インストール成功
```

ただし、初回失敗時の詳細エラーを確認していないため、原因は完全には断定できない。

次回は以下を確認する。

```text
Systems Manager
↓
Run Command
↓
Command history
↓
実行したコマンド
↓
Targets and outputs
↓
対象 EC2
↓
Output / Error
```

---

## 19. CloudWatch Agent のファイルが見えなかった

最初の Run Command 実行後、以下を確認した。

```bash
ls -l /opt/aws/amazon-cloudwatch-agent/bin/
```

しかし、ファイルが表示されなかった。

原因：

```text
CloudWatch Agent のインストールが失敗していた
```

対処：

```text
Run Command の実行結果を確認
↓
S3 Gateway Endpoint を追加
↓
Run Command を再実行
↓
インストール成功
```

---

## 20. vi の保存方法で迷った

誤って以下を考えた。

```text
:eq
```

正しい保存・終了方法：

```text
Esc
↓
:wq
↓
Enter
```

意味：

```text
:w
→ 保存

:q
→ 終了
```

保存せず終了する場合：

```text
:q!
```

---

# 今回の重要ポイント

## 21. CloudWatch Logs が EC2 を直接見に来るわけではない

```text
CloudWatch Logs
↓
EC2 内を見に来る
```

ではない。

正しくは以下。

```text
EC2 内の CloudWatch Agent
↓
ログファイルを読む
↓
CloudWatch Logs へ送る
```

---

## 22. 暗記より役割を理解する

正式名称や長いコマンドをすべて暗記する必要はない。

まずは役割を理解する。

```text
SSM Agent
→ AWS から EC2 を操作する

Run Command
→ Systems Manager 経由で EC2 に命令を送る

CloudWatch Agent
→ EC2 内のログを CloudWatch Logs へ送る

IAM Role
→ EC2 に必要な権限を渡す

VPC Endpoint
→ Private EC2 から AWS サービスへ通信する経路を作る
```

正式名称、JSON、長いコマンドは、必要なときに手順書を確認する。

