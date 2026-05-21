# EC2 IAM Role S3 Access Lab

## 概要

EC2インスタンスにIAM Roleを付与し、アクセスキーをEC2内に保存せずに、AWS CLIからS3へアクセスする実践。

S3にアップロードした `test.txt` を、EC2からIAM Role経由で取得できることを確認した。

---

## 実践内容

今回実施した内容は以下。

```text
S3バケット作成
test.txtアップロード
EC2用IAM Role作成
AmazonS3ReadOnlyAccess付与
EC2起動時にIAM Roleを付与
SSH接続
AWS CLI v2インストール
IAM Roleで認証確認
S3バケット一覧確認
S3からEC2へファイルコピー
catで中身確認
リソース削除
```

---

## 構成イメージ

```text
EC2
│
├── IAM Role
│   └── AmazonS3ReadOnlyAccess
│
└── AWS CLI
    └── S3へアクセス

S3
└── test.txt
```

---

## 重要ポイント

```text
EC2にIAM Roleを付けると、
aws configureをしなくてもAWS CLIでS3を操作できる
```

EC2内にアクセスキーを保存せず、IAM Roleを使ってAWSサービスへアクセスすることを確認した。

---

## 使用した主なコマンド

```bash
aws sts get-caller-identity
aws s3 ls
aws s3 ls s3://バケット名/
aws s3 cp s3://バケット名/test.txt .
cat test.txt
```

---

## ドキュメント

```text
docs/setup-steps.md
→ セットアップ手順

docs/setup-reasons.md
→ なぜその設定にしたか

docs/ec2-after-launch-flow.md
→ EC2起動後の作業フロー

docs/commands-used.md
→ 使用コマンドまとめ

docs/learning-notes.md
→ 学びのまとめ

docs/troubleshooting.md
→ トラブルシューティング

docs/security-points.md
→ セキュリティ面のポイント

docs/cleanup-checklist.md
→ 後片付けチェックリスト
```

---

## 今回の学び

```text
IAM Roleは、EC2などにAWS操作権限を渡す仕組み
S3は、AWS上のオブジェクトストレージ
EC2からS3へアクセスするには、権限が必要
アクセスキーを置かずにIAM Roleで認証できる
aws s3 lsは一覧表示
aws s3 cpはファイルコピー
```

---

## 後片付け

実践後、課金を抑えるために以下を削除した。

```text
EC2インスタンス
S3内のtest.txt
S3バケット
IAM Role
```

---

## まとめ

今回の実践により、EC2にIAM Roleを付けることで、アクセスキーを保存せずにAWS CLIからS3へアクセスできることを確認した。

AWSでは、サーバ内に認証情報を直接置くのではなく、IAM Roleを使って必要な権限を渡すことが重要。
