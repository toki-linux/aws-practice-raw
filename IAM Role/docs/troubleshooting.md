# トラブルシューティング：EC2 + IAM Role + S3

## 目的

このドキュメントでは、EC2からIAM Roleを使ってS3へアクセスする実践中に起きたエラーや確認ポイントを整理する。

---

## 1. awsコマンドが見つからない

### 状況

EC2へSSH接続後、AWS CLIが入っているか確認した。

```bash
aws --version
```

結果。

```text
Command 'aws' not found
```

### 原因

作成したUbuntu環境には、AWS CLIが最初から入っていなかった。

### 対応

AWS CLI v2をインストールした。

```bash
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

確認。

```bash
aws --version
```

---

## 2. aptでawscliをインストールできない

### 状況

aptでAWS CLIを入れようとした。

```bash
sudo apt install awscli
```

結果。

```text
Package awscli is not available
E: Package 'awscli' has no installation candidate
```

### 原因

パッケージ名は `awscli` で合っているが、この環境ではaptのリポジトリから取得できなかった。

### 対応

aptでのインストールを深追いせず、AWS CLI v2を手動インストールした。

```bash
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## 3. S3のファイルがEC2上に存在しない

### 状況

`cat test.txt` を実行したところ、ファイルが存在しないと言われた。

```bash
cat test.txt
```

### 原因

S3上のファイル一覧を見る `aws s3 ls` は実行していたが、S3からEC2へコピーする `aws s3 cp` が実行できていなかった。

つまり、EC2のカレントディレクトリに `test.txt` が存在していなかった。

### 理解したこと

```text
aws s3 ls
→ S3上の一覧を見るだけ
→ EC2上にファイルは作られない

aws s3 cp
→ S3からEC2へファイルをコピーする
→ EC2上にファイルが作られる
```

### 対応

S3からEC2へファイルをコピーした。

```bash
aws s3 cp s3://バケット名/test.txt .
```

その後、EC2上にファイルがあるか確認した。

```bash
ls -l
```

中身を確認した。

```bash
cat test.txt
```

---

## 4. バケット名をそのまま打ってしまう可能性

### 注意点

以下のようなコマンド例がある。

```bash
aws s3 cp s3://バケット名/test.txt .
```

この `バケット名` は、そのまま入力する文字ではなく、自分が作成したS3バケット名に置き換える。

### 例

バケット名が以下の場合。

```text
toki-iam-role-practice-20260521
```

実行するコマンドは以下。

```bash
aws s3 cp s3://toki-iam-role-practice-20260521/test.txt .
```

---

## 5. バケット内のファイル名が違う可能性

### 状況

`aws s3 cp` でファイルが見つからない場合、S3上のファイル名が想定と違う可能性がある。

### 確認コマンド

```bash
aws s3 ls s3://バケット名/
```

### よくある違い

```text
test.txt
test.txt.txt
Test.txt
フォルダ/test.txt
```

S3のオブジェクト名は大文字・小文字を区別する。

そのため、S3上に表示された名前と同じ名前を指定する必要がある。

---

## 6. IAM Roleが効いているか確認する

### 確認コマンド

```bash
aws sts get-caller-identity
```

### 期待する結果

```text
assumed-role/EC2-S3-ReadOnly-Role/...
```

このように、作成したRole名が表示されれば、EC2がIAM Role経由でAWSにアクセスしている。

### もしエラーになる場合に見るポイント

```text
EC2にIAM Roleを付けたか
EC2作成時にIAMインスタンスプロフィールを設定したか
Roleの信頼されたエンティティがEC2になっているか
RoleにS3読み取り権限が付いているか
```

---

## 7. aws configureは不要

### 注意点

今回の実践では、`aws configure` は実行しない。

```bash
aws configure
```

### 理由

今回確認したいことは、アクセスキーを使わずにIAM RoleでAWSへアクセスすることだから。

```text
aws configure
→ アクセスキーをEC2内に保存する

IAM Role
→ アクセスキーをEC2内に置かずにAWSへアクセスする
```

今回の目的と逆になるため、`aws configure` は使わない。

---

## トラブル時の確認順

うまくいかない場合は、以下の順で確認する。

```text
1. AWS CLIが入っているか
2. EC2にIAM Roleが付いているか
3. aws sts get-caller-identityでRoleが見えるか
4. aws s3 lsでバケット一覧が見えるか
5. aws s3 ls s3://バケット名/ でtest.txtが見えるか
6. aws s3 cpでEC2へコピーできているか
7. ls -lでEC2上にtest.txtがあるか
8. cat test.txtで中身を確認できるか
```

---

## 今回のつまずきから得た学び

今回のつまずきで、以下を理解できた。

```text
S3上にファイルがあることと、
EC2上にファイルがあることは別
```

S3上のファイルをEC2で読むには、まずコピーが必要。

```bash
aws s3 cp s3://バケット名/test.txt .
```

その後に、EC2上で確認する。

```bash
cat test.txt
```
