# 学びのまとめ：EC2 + IAM Role + S3

## 今回の実践で学んだこと

EC2インスタンスにIAM Roleを付与することで、EC2内にアクセスキーを保存せずに、AWS CLIからS3へアクセスできることを確認した。

今回の一番大事な学びは以下。

```text
EC2にIAM Roleを付けると、
aws configureをしなくてもAWS CLIでS3を操作できる
```

---

## IAM Roleとは

IAM Roleは、AWSサービスやEC2などに一時的な権限を渡すための仕組み。

今回の場合は、EC2に対してS3を読み取る権限を渡した。

```text
誰に：EC2
何を：S3の読み取り
どうやって：IAM Roleを付与
```

つまり、今回のRoleは以下の意味を持つ。

```text
このEC2インスタンスは、S3を読み取ってよい
```

---

## IAM UserとIAM Roleの違い

IAM Userは、人やアプリケーションが直接使う認証情報として利用される。

一方、IAM Roleは、EC2などのAWSリソースに権限を渡すために使う。

```text
IAM User
→ 人や固定的な利用者に権限を付けるイメージ

IAM Role
→ EC2などに一時的な権限を渡すイメージ
```

今回の実践では、EC2内にアクセスキーを置かず、IAM Roleを利用した。

---

## aws configureを使わなかった理由

今回、EC2内で `aws configure` は使わなかった。

理由は、アクセスキーを使わずにIAM Roleで認証することを確認したかったため。

```text
aws configureを使う
→ アクセスキーをEC2内に設定する

IAM Roleを使う
→ アクセスキーをEC2内に置かずにAWSサービスへアクセスする
```

今回確認したかったことは以下。

```text
EC2にIAM Roleを付ければ、
アクセスキーなしでAWS CLIからS3へアクセスできる
```

---

## S3を使った理由

今回、IAM Roleの動作確認先としてS3を使った。

理由は、ファイルを置いて確認しやすく、AWS CLIでの操作も分かりやすいため。

```text
S3バケット一覧を見る
S3バケット内のファイルを見る
S3からEC2へファイルをコピーする
コピーしたファイルの中身を確認する
```

この流れによって、EC2に付与したIAM Roleが実際に使われていることを確認できた。

---

## EBSとS3の違い

EC2には通常、EBSがルートボリュームとして付いている。

EBSはEC2に接続して使うディスク。

一方で、S3はAWS上の外部ストレージ。

```text
EBS
→ EC2に接続して使うディスク
→ OS、設定ファイル、アプリなどを置く

S3
→ AWS上のオブジェクトストレージ
→ ファイル、ログ、バックアップ、画像などを置く
```

今回S3を使ったのは、EC2から別のAWSサービスへアクセスする練習をするため。

---

## AmazonS3ReadOnlyAccessを使った理由

今回のIAM Roleには `AmazonS3ReadOnlyAccess` を付与した。

理由は、今回必要だった操作がS3の読み取りだけだったため。

今回やったこと。

```text
S3バケット一覧を見る
S3バケット内のファイルを見る
S3からファイルを取得する
```

今回やらなかったこと。

```text
S3へファイルをアップロードする
S3のファイルを削除する
S3バケットを作成・削除する
```

そのため、書き込みや削除の権限は不要だった。

---

## aws sts get-caller-identityの意味

`aws sts get-caller-identity` は、現在AWS CLIがどの認証情報でAWSにアクセスしているか確認するコマンド。

今回このコマンドを使うことで、EC2がIAM Role経由で認証されていることを確認した。

```bash
aws sts get-caller-identity
```

期待する結果。

```text
assumed-role/EC2-S3-ReadOnly-Role/...
```

`assumed-role` と表示されることで、IAM Roleを引き受けてAWSにアクセスしていることが分かる。

---

## aws s3 lsとaws s3 cpの違い

今回つまずいたポイントとして、`ls` と `cp` の違いがある。

```bash
aws s3 ls s3://バケット名/
```

これはS3上の一覧を見るだけ。

EC2上にファイルは作られない。

一方で、以下はS3からEC2へファイルをコピーする。

```bash
aws s3 cp s3://バケット名/test.txt .
```

整理すると以下。

```text
aws s3 ls
→ S3上のファイル一覧を見る

aws s3 cp
→ S3上のファイルをEC2へコピーする
```

そのため、`aws s3 ls` だけを実行した状態で `cat test.txt` をしても、EC2上にファイルがないためエラーになる。

---

## 今回理解できた全体像

今回の全体像は以下。

```text
S3にtest.txtを置く
↓
EC2用IAM Roleを作る
↓
RoleにS3読み取り権限を付ける
↓
EC2にRoleを付ける
↓
EC2にSSH接続する
↓
AWS CLIを入れる
↓
aws configureはしない
↓
IAM Role経由でS3へアクセスする
↓
S3からtest.txtを取得する
```

---

## 今回の重要ポイント

```text
EC2にIAM Roleを付けると、
EC2内にアクセスキーを保存しなくてもAWSサービスを操作できる
```

この考え方は、AWSを安全に使う上で重要。

アクセスキーを直接サーバ内に置くのではなく、必要な権限をIAM RoleとしてEC2に渡す。
