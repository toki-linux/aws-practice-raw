# screenshots

このディレクトリには、CloudWatch Logs実践の確認用スクリーンショットを保存します。

## 保存するスクリーンショット例

- EC2インスタンス作成画面
- EC2にIAM Roleが付与されている画面
- CloudWatch Logsのロググループ画面
- `/ec2/nginx/access` のログストリーム画面
- access.logのログイベント表示画面
- `/ec2/nginx/error` のロググループ画面

---

## 注意点

スクリーンショットをGitHubにアップロードする前に、以下の情報が写っていないか確認します。

- AWSアカウントID
- メールアドレス
- 不要なパブリックIP
- 秘密情報
- 個人情報
- 不要なリソース名

---

## 今回確認したい画面

### CloudWatch Logsのロググループ

```text
/ec2/nginx/access
/ec2/nginx/error
```

### ログストリーム

今回の設定では、ログストリーム名にEC2のインスタンスIDを使用しました。

```json
"log_stream_name": "{instance_id}"
```

### ログイベント

以下のようなnginxアクセスログが表示されていれば成功です。

```text
GET / HTTP/1.1" 200
GET /notfound HTTP/1.1" 404
```
