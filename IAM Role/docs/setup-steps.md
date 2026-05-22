# EC2 + IAM Role + S3 実践：セットアップ手順

## 目的

EC2インスタンスにIAM Roleを付与し、EC2内にアクセスキーを保存せずに、AWS CLIからS3へアクセスできることを確認する。

今回の実践では、以下を行う。

- S3バケットを作成する
- S3にテストファイルをアップロードする
- EC2用のIAM Roleを作成する
- IAM RoleにS3読み取り権限を付与する
- EC2インスタンス作成時にIAM Roleを付与する
- EC2へSSH接続する
- EC2内にAWS CLIをインストールする
- EC2からS3上のファイルを取得する

---

## 1. S3バケットを作成する

AWSマネジメントコンソールでS3を開き、バケットを作成した。

設定内容は以下。

```text
バケットタイプ：汎用
バケット名前空間：グローバル名前空間
リージョン：EC2と同じリージョン
パブリックアクセス：すべてブロック
バージョニング：無効
暗号化：デフォルト設定
2. test.txtをアップロードする

作成したS3バケットに、動作確認用の test.txt をアップロードした。

ファイル内容の例。

hello from s3

S3上の構成は以下。

S3バケット
└── test.txt
3. IAM Roleを作成する

IAMからEC2用のIAM Roleを作成した。

設定内容は以下。

信頼されたエンティティタイプ：AWSのサービス
ユースケース：EC2

付与した許可ポリシーは以下。

AmazonS3ReadOnlyAccess

ロール名の例。

EC2-S3-ReadOnly-Role

このIAM Roleにより、EC2がS3を読み取る権限を持てるようにした。

4. EC2インスタンスを作成する

EC2インスタンスを作成した。

設定内容は以下。

AMI：Ubuntu
インスタンスタイプ：低コスト系
キーペア：SSH接続用の既存キーペア
セキュリティグループ：SSHのみ許可
IAMインスタンスプロフィール：作成したEC2用IAM Role

今回の目的はS3アクセス確認のため、HTTPは開放しない。

許可した通信：SSH 22番
許可しない通信：HTTP 80番
5. EC2へSSH接続する

作成したEC2インスタンスへSSH接続した。

ssh -i キー名.pem ubuntu@パブリックIP
6. AWS CLIの有無を確認する

EC2内でAWS CLIが入っているか確認した。

aws --version

最初はAWS CLIが入っていなかったため、以下のようなエラーが表示された。

Command 'aws' not found
7. AWS CLI v2をインストールする

apt install awscli ではインストールできなかったため、AWS CLI v2を手動でインストールした。

必要なパッケージを入れる。

sudo apt install -y unzip curl

AWS CLI v2のzipファイルをダウンロードする。

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

zipファイルを展開する。

unzip awscliv2.zip

インストールする。

sudo ./aws/install

インストール確認。

aws --version
8. IAM Roleで認証されていることを確認する

EC2内で以下を実行した。

aws sts get-caller-identity

ここで assumed-role/EC2-S3-ReadOnly-Role/... のような表示が出ることを確認した。

これにより、EC2内にアクセスキーを設定しなくても、IAM Role経由でAWSにアクセスできていることが確認できた。

9. S3バケット一覧を確認する

EC2内からS3バケット一覧を確認した。

aws s3 ls

作成したS3バケットが表示されることを確認した。

10. S3バケット内のファイルを確認する

作成したバケット内のオブジェクトを確認した。

aws s3 ls s3://バケット名/

test.txt が表示されることを確認した。

11. S3からEC2へファイルをコピーする

S3上の test.txt を、EC2のカレントディレクトリへコピーした。

aws s3 cp s3://バケット名/test.txt .
12. コピーしたファイルの中身を確認する

EC2上にコピーされたファイルを確認した。

cat test.txt

表示例。

hello from s3

これにより、EC2からIAM Roleを使ってS3上のファイルを取得できることを確認した。

13. 後片付け

課金を抑えるため、実践後に以下を削除した。

EC2インスタンス
S3内のtest.txt
S3バケット
作成したIAM Role

IAM Role自体は基本的に課金対象ではないが、今回の練習用に作成したものなので削除した。
