# 使用したコマンドまとめ

## 目的

このドキュメントでは、EC2からIAM Roleを使ってS3へアクセスする実践で使用したコマンドを整理する。

---

## SSH接続

EC2へSSH接続する。

```bash
ssh -i キー名.pem ubuntu@パブリックIP
コマンドの意味
ssh
→ リモートサーバへ接続するコマンド

-i キー名.pem
→ SSH接続に使う秘密鍵を指定する

ubuntu
→ Ubuntu AMIのログインユーザー

パブリックIP
→ EC2インスタンスのパブリックIPアドレス
AWS CLIの確認

AWS CLIが入っているか確認する。

aws --version
目的
EC2内でawsコマンドが使えるか確認する

AWS CLIが入っていない場合は、以下のような表示になる。

Command 'aws' not found
aptでAWS CLIをインストールしようとしたコマンド
sudo apt install awscli
結果

今回の環境では、以下のように表示されてインストールできなかった。

Package awscli is not available
E: Package 'awscli' has no installation candidate

そのため、AWS CLI v2を手動でインストールした。

必要パッケージのインストール

AWS CLI v2のインストールに必要なパッケージを入れる。

sudo apt install -y unzip curl
コマンドの意味
sudo
→ 管理者権限で実行する

apt install
→ パッケージをインストールする

-y
→ 確認メッセージに自動でyesと答える

unzip
→ zipファイルを展開するためのコマンド

curl
→ URLからファイルをダウンロードするためのコマンド
AWS CLI v2のダウンロード

AWS CLI v2のzipファイルをダウンロードする。

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
コマンドの意味
curl
→ URLにアクセスしてデータを取得する

-o "awscliv2.zip"
→ ダウンロードした内容をawscliv2.zipという名前で保存する
zipファイルの展開

ダウンロードしたzipファイルを展開する。

unzip awscliv2.zip
コマンドの意味
unzip
→ zipファイルを展開する

awscliv2.zip
→ 展開するzipファイル

展開すると、aws ディレクトリが作成される。

AWS CLI v2のインストール

展開されたインストーラーを実行する。

sudo ./aws/install
コマンドの意味
sudo
→ 管理者権限で実行する

./aws/install
→ カレントディレクトリ配下のaws/installを実行する
AWS CLIインストール後の確認
aws --version
目的
AWS CLIがインストールされ、awsコマンドが使えることを確認する
IAM Roleで認証されているか確認
aws sts get-caller-identity
目的

現在のAWS CLIが、どの認証情報でAWSにアクセスしているか確認する。

期待する結果。

assumed-role/EC2-S3-ReadOnly-Role/...

このように、作成したIAM Role名が表示されれば、EC2がIAM Roleを使ってAWSにアクセスしていると分かる。

コマンドの意味
aws
→ AWS CLIを実行する

sts
→ AWS Security Token Service

get-caller-identity
→ 現在の呼び出し元の情報を表示する
S3バケット一覧を確認
aws s3 ls
目的

EC2からS3バケット一覧を取得できるか確認する。

コマンドの意味
aws s3
→ AWS CLIでS3を操作する

ls
→ 一覧を表示する

このコマンドで作成したS3バケットが表示されれば、S3への読み取りアクセスができている。

S3バケット内のファイルを確認
aws s3 ls s3://バケット名/
目的

指定したS3バケット内に、アップロードした test.txt が存在するか確認する。

コマンドの意味
s3://バケット名/
→ 対象のS3バケットを指定する

表示例。

test.txt
S3からEC2へファイルをコピー
aws s3 cp s3://バケット名/test.txt .
目的

S3上の test.txt を、EC2のカレントディレクトリへコピーする。

コマンドの意味
aws s3 cp
→ S3とローカル間でファイルをコピーする

s3://バケット名/test.txt
→ コピー元のS3オブジェクト

.
→ コピー先を現在のディレクトリにする
カレントディレクトリのファイルを確認
ls -l
目的

S3からコピーした test.txt が、EC2上に存在するか確認する。

コマンドの意味
ls
→ ファイル一覧を表示する

-l
→ 詳細表示にする
ファイルの中身を確認
cat test.txt
目的

S3からコピーしたファイルの中身を確認する。

表示例。

hello from s3
コマンドの意味
cat
→ ファイルの中身を表示する

test.txt
→ 表示する対象ファイル
lsとcpの違い

今回の実践で重要だった違い。

aws s3 ls s3://バケット名/

これは、S3上のファイル一覧を見るだけ。

aws s3 cp s3://バケット名/test.txt .

これは、S3上のファイルをEC2へコピーする。

整理すると以下。

aws s3 ls
→ 一覧を見るだけ
→ EC2上にファイルは作られない

aws s3 cp
→ ファイルをコピーする
→ EC2上にファイルが作られる

そのため、aws s3 ls だけを実行した状態で以下を実行しても、ファイルは存在しない。

cat test.txt

cat test.txt を実行する前に、aws s3 cp でEC2へコピーする必要がある。

今回使わなかったコマンド
aws configure
aws configure

今回はこのコマンドを使わなかった。

理由は、IAM Roleによる認証を確認することが目的だったから。

aws configure
→ アクセスキーをEC2内に設定する

IAM Role
→ アクセスキーをEC2内に置かずにAWSサービスへアクセスする

今回確認したかったこと。

EC2にIAM Roleを付ければ、aws configureなしでS3へアクセスできる
コマンド実行順まとめ
ssh -i キー名.pem ubuntu@パブリックIP
aws --version
sudo apt install awscli
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
aws sts get-caller-identity
aws s3 ls
aws s3 ls s3://バケット名/
aws s3 cp s3://バケット名/test.txt .
ls -l
cat test.txt
このコマンド群で確認できたこと
AWS CLIをEC2にインストールできた
EC2がIAM Roleで認証されていることを確認できた
EC2からS3バケット一覧を確認できた
EC2からS3バケット内のファイルを確認できた
S3からEC2へファイルをコピーできた
コピーしたファイルの中身を確認できた

今回の一番大事な確認ポイント。

EC2にIAM Roleを付けると、aws configureをしなくてもAWS CLIでS3を操作できる
