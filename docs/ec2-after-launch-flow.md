# EC2起動後の作業フロー

## 目的

このドキュメントでは、EC2インスタンスを起動してから、IAM Role経由でS3上のファイルを取得するまでの流れを整理する。

今回の目的は、EC2内にアクセスキーを保存せず、IAM Roleを使ってAWS CLIからS3へアクセスできることを確認すること。

---

## 全体の流れ

```text
EC2インスタンス起動
↓
SSH接続
↓
AWS CLIの有無を確認
↓
AWS CLI v2をインストール
↓
IAM Roleで認証されていることを確認
↓
S3バケット一覧を確認
↓
S3バケット内のファイルを確認
↓
S3からEC2へファイルをコピー
↓
コピーしたファイルの中身を確認
1. EC2インスタンスを起動する

事前に作成したIAM Roleを付与して、EC2インスタンスを起動した。

今回のEC2では、以下の設定にした。

AMI：Ubuntu
セキュリティグループ：SSHのみ許可
IAMインスタンスプロフィール：EC2用IAM Role

今回の目的はS3アクセス確認のため、HTTPは開放しなかった。

2. SSH接続する

EC2インスタンスが起動したら、SSHで接続した。

ssh -i キー名.pem ubuntu@パブリックIP

UbuntuのEC2インスタンスでは、ログインユーザーは基本的に ubuntu を使う。

3. AWS CLIが入っているか確認する

EC2に接続後、まずAWS CLIが入っているか確認した。

aws --version

結果として、AWS CLIは入っていなかった。

Command 'aws' not found

そのため、AWS CLIをインストールする必要があった。

4. aptでawscliを入れようとしたが失敗

最初に、aptでAWS CLIを入れようとした。

sudo apt install awscli

しかし、以下のように表示された。

Package awscli is not available
E: Package 'awscli' has no installation candidate

パッケージ名は awscli で合っているが、この環境ではaptからインストールできなかった。

そのため、AWS CLI v2を公式手順に近い形で手動インストールした。

5. AWS CLI v2をインストールする

AWS CLI v2のインストールに必要なパッケージを入れた。

sudo apt install -y unzip curl

AWS CLI v2のzipファイルをダウンロードした。

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

zipファイルを展開した。

unzip awscliv2.zip

インストーラーを実行した。

sudo ./aws/install

インストール後、バージョンを確認した。

aws --version

これで aws コマンドが使えるようになった。

6. aws configureは実行しない

今回、EC2内で aws configure は実行しなかった。

理由は、アクセスキーをEC2内に保存せず、IAM Roleで認証することを確認するため。

今回やること：
EC2にIAM Roleを付けてAWS CLIを使う

今回やらないこと：
EC2内にアクセスキーを設定する

IAM Roleが正しく付与されていれば、aws configure をしなくてもAWS CLIからS3へアクセスできる。

7. IAM Roleで認証されていることを確認する

次に、EC2がどの権限でAWSにアクセスしているか確認した。

aws sts get-caller-identity

このコマンドにより、現在AWS CLIが使っている認証情報を確認できる。

期待する結果は、作成したIAM Roleが表示されること。

assumed-role/EC2-S3-ReadOnly-Role/...

このような表示が出れば、EC2がIAM Roleを使ってAWSにアクセスしていると分かる。

8. S3バケット一覧を確認する

次に、EC2からS3バケット一覧を確認した。

aws s3 ls

ここで、事前に作成したS3バケットが表示されることを確認した。

この時点で、EC2からS3に対して読み取りアクセスできていることが分かる。

9. S3バケット内のファイルを確認する

次に、バケット内にアップロードしたファイルを確認した。

aws s3 ls s3://バケット名/

ここで、test.txt が表示されることを確認した。

test.txt

これにより、S3バケット内のオブジェクト一覧をEC2から取得できていることが確認できた。

10. S3からEC2へファイルをコピーする

S3上の test.txt を、EC2のカレントディレクトリへコピーした。

aws s3 cp s3://バケット名/test.txt .

最後の . は、現在いるディレクトリを意味する。

つまり、このコマンドは以下の意味。

S3上のtest.txtを、EC2の今いる場所にコピーする
11. EC2上にコピーされたことを確認する

コピー後、EC2上に test.txt が存在するか確認した。

ls -l

test.txt が表示されれば、S3からEC2へのコピーは成功。

12. ファイルの中身を確認する

コピーしたファイルの中身を確認した。

cat test.txt

表示例。

hello from s3

これにより、EC2からIAM Roleを使ってS3上のファイルを取得し、中身を確認できた。

13. 今回つまずいたポイント

今回、aws s3 cp を実行するつもりが、aws s3 ls になっていたため、EC2のカレントディレクトリに test.txt が存在しない状態になった。

その状態で以下を実行した。

cat test.txt

すると、EC2上にファイルがないため、エラーになった。

ここで理解したこと。

aws s3 ls
→ S3上の一覧を見るだけ

aws s3 cp
→ S3からEC2へファイルをコピーする

ls ではファイルはEC2上に作成されない。

14. 実践後の削除

動作確認後、課金を抑えるために作成したリソースを削除した。

削除したもの。

EC2インスタンス
S3内のtest.txt
S3バケット
作成したIAM Role

IAM Role自体は基本的に課金対象ではないが、練習用に作成したものなので削除した。

この実践で確認できたこと
EC2にIAM Roleを付ける
↓
EC2内でaws configureをしない
↓
AWS CLIを実行する
↓
IAM Roleの権限でS3にアクセスできる
↓
S3上のファイルをEC2へコピーできる

今回の一番重要なポイントは以下。

EC2にIAM Roleを付けると、アクセスキーをEC2内に保存しなくてもAWS CLIでS3を操作できる
