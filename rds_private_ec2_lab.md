# Private EC2からAmazon RDS for MySQLへ接続する実践

## 1. 概要

Private Subnetに配置したEC2から、Amazon RDS for MySQLへ接続する環境を構築した。

今回の目的は、RDSを起動することだけではない。

```text
Private EC2
↓
RDSへ接続
↓
データベースを作成
↓
テーブルを作成
↓
データを保存
↓
保存したデータを取得
```

まで実際に確認すること。

さらに、RDS用Security Groupのルールを意図的に削除し、通信失敗を再現した。

---

# 2. 今回の構成

```text
自分のPC
↓
AWS Systems Manager Session Manager
↓
SSM系Interface Endpoint
↓
Private EC2
↓ TCP 3306
Amazon RDS for MySQL
```

VPC内部。

```text
VPC：10.0.0.0/16

├── Private Subnet-A
│   ├── Private EC2
│   └── RDSの配置候補
│
└── Private Subnet-C
    └── RDSの配置候補
```

RDSへ接続するため、EC2とRDSは同じVPC内に配置した。

---

# 3. 今回作成したリソース

```text
VPC
Private Subnet × 2
Route Table
Private EC2
EC2用Security Group
SSM Endpoint用Security Group
RDS用Security Group
SSM用IAM Role
SSM系Interface Endpoint × 3
S3 Gateway Endpoint
DB Subnet Group
Amazon RDS for MySQL
```

---

# 4. 異なるAZにSubnetを2つ作る理由

RDSでは、DB Subnet Groupへ異なるAZのSubnetを登録する。

```text
DB Subnet Group
├── Private Subnet-A：ap-northeast-1a
└── Private Subnet-C：ap-northeast-1c
```

今回使用したのはSingle-AZ構成。

```text
Single-AZ
→ 実際に起動するRDSは1台
```

そのため、2つ目のAZに予備DBが自動で起動するわけではない。

異なるAZにSubnetを作る理由は、

```text
将来、別AZにもRDSを配置できる候補地を確保する
```

ため。

Multi-AZ構成にした場合は、別AZにStandby DBを用意できる。

---

# 5. Security Groupの設計

## 5.1 EC2用Security Group

名前。

```text
private-sg
```

Inbound Rule。

```text
なし
```

Outbound Rule。

```text
すべてのトラフィック
送信先：0.0.0.0/0
```

Private EC2へはSSHではなく、Session Managerで接続する。

---

## 5.2 SSM Endpoint用Security Group

名前。

```text
ssm-sg
```

Inbound Rule。

```text
Type：HTTPS
Protocol：TCP
Port：443
Source：private-sg
```

Private EC2からSSM系Interface EndpointへHTTPS通信できるようにする。

---

## 5.3 RDS用Security Group

名前。

```text
rds-sg
```

Inbound Rule。

```text
Type：MySQL/Aurora
Protocol：TCP
Port：3306
Source：private-sg
```

これにより、

```text
private-sgを持つEC2
↓ TCP 3306
rds-sgを持つRDS
```

の通信だけを許可する。

Internet全体へRDSを公開する必要はない。

---

# 6. IAM Role

Private EC2へ付与したIAM Policy。

```text
AmazonSSMManagedInstanceCore
```

目的。

```text
EC2内のSSM Agentが
AWS Systems Managerと通信するため
```

---

# 7. SSM系Interface Endpoint

Private EC2にはPublic IPを付けず、NAT Gatewayも作らなかった。

Session Managerで接続するため、次のInterface Endpointを作成した。

```text
com.amazonaws.ap-northeast-1.ssm

com.amazonaws.ap-northeast-1.ssmmessages

com.amazonaws.ap-northeast-1.ec2messages
```

特徴。

```text
Subnet内にEndpoint用ENIが作られる
Private IPを持つ
Security Groupを付ける
Private DNSを使用する
```

---

# 8. DB Subnet Group

RDSを配置してよいSubnet候補をまとめるため、DB Subnet Groupを作成した。

例。

```text
toki-rds-subnet-group
```

登録したSubnet。

```text
Private Subnet-A
Private Subnet-C
```

イメージ。

```text
DB Subnet Group
├── Private Subnet-A
└── Private Subnet-C
```

DB Subnet Groupは、

```text
RDSを置いてよいネットワーク上の候補地
```

を指定するための設定。

---

# 9. RDSをCLIで作成した理由

本来は、AWSマネジメントコンソールのRDS作成画面から設定する予定だった。

しかし、DBインスタンスタイプの候補は表示されるものの、すべてグレーアウトして選択できなかった。

CloudShellから確認したところ、

```text
db.t4g.micro
db.t3.micro
```

は利用可能な候補として表示された。

そのため、RDS本体だけAWS CLIで作成した。

```text
AWSマネジメントコンソール
→ プルダウンで設定を選ぶ

AWS CLI
→ オプションで同じ設定を指定する
```

CLIの長いコマンドを丸暗記する必要はない。

重要なのは、

```text
どの設定を指定しているか
```

を理解すること。

---

# 10. RDS作成前の確認

## 10.1 RDS用Security Group IDを確認

CloudShellで実行。

```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=rds-sg" \
  --query "SecurityGroups[].{Name:GroupName,ID:GroupId,VPC:VpcId}" \
  --output table
```

結果例。

```text
sg-xxxxxxxxxxxxxxxxx
```

---

## 10.2 DB Subnet Groupを確認

```bash
aws rds describe-db-subnet-groups \
  --db-subnet-group-name toki-rds-subnet-group \
  --query "DBSubnetGroups[0].{Name:DBSubnetGroupName,VPC:VpcId,Subnets:Subnets[].SubnetIdentifier}" \
  --output json
```

確認したこと。

```text
Subnetが2つ登録されている
異なるAZに所属している
RDS用SGと同じVPCにある
```

---

# 11. CLIでRDSを作成

## 11.1 パスワードを一時変数へ保存

パスワードをコマンド履歴へ直接残さないため、非表示で入力した。

```bash
read -s DB_PASSWORD
```

入力しても、画面には表示されない。

---

## 11.2 RDS用Security Group IDを変数へ保存

```bash
RDS_SG_ID="sg-xxxxxxxxxxxxxxxxx"
```

確認。

```bash
echo "$RDS_SG_ID"
```

---

## 11.3 RDS for MySQLを作成

```bash
aws rds create-db-instance \
  --db-instance-identifier toki-rds-mysql \
  --engine mysql \
  --engine-version 8.4.8 \
  --db-instance-class db.t4g.micro \
  --allocated-storage 20 \
  --storage-type gp3 \
  --master-username admin \
  --master-user-password "$DB_PASSWORD" \
  --port 3306 \
  --db-subnet-group-name toki-rds-subnet-group \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --backup-retention-period 0 \
  --no-multi-az \
  --no-publicly-accessible \
  --no-deletion-protection
```

指定した内容。

```text
DBエンジン
→ MySQL

DBインスタンス識別子
→ toki-rds-mysql

DBインスタンスタイプ
→ db.t4g.micro

ストレージ
→ gp3 / 20 GiB

マスターユーザー
→ admin

接続ポート
→ TCP 3306

DB Subnet Group
→ toki-rds-subnet-group

Security Group
→ rds-sg

可用性
→ Single-AZ

Public access
→ なし

削除保護
→ なし
```

---

## 11.4 パスワード変数を削除

```bash
unset DB_PASSWORD
```

---

# 12. RDSの状態確認

```bash
aws rds describe-db-instances \
  --db-instance-identifier toki-rds-mysql \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text
```

作成中。

```text
creating
```

作成完了。

```text
available
```

---

# 13. RDS Endpointを確認

CloudShellで実行。

```bash
aws rds describe-db-instances \
  --db-instance-identifier toki-rds-mysql \
  --query "DBInstances[0].{Endpoint:Endpoint.Address,Port:Endpoint.Port,User:MasterUsername,Status:DBInstanceStatus}" \
  --output table
```

確認した情報。

```text
Endpoint
Port
MasterUsername
Status
```

RDS Endpointは、EC2からRDSへ接続するためのDNS名。

例。

```text
toki-rds-mysql.xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com
```

---

# 14. S3 Gateway Endpointを追加

Private EC2へMySQLクライアントをインストールしようとしたが、処理が進まなかった。

実行したコマンド。

```bash
sudo dnf install mariadb105
```

原因。

```text
Private EC2
├── Public IPなし
├── NAT Gatewayなし
└── S3への経路なし
```

Session ManagerでEC2へ入れることと、パッケージを取得できることは別。

Amazon Linux 2023のパッケージを取得するため、S3 Gateway Endpointを追加した。

サービス名。

```text
com.amazonaws.ap-northeast-1.s3
```

タイプ。

```text
Gateway
```

関連付け先。

```text
Private EC2が所属するSubnet用のRoute Table
```

追加後、Route TableにS3向けルートが作成された。

```text
Destination：pl-xxxxxxxx
Target：vpce-xxxxxxxx
```

---

# 15. MySQLクライアントをインストール

Session ManagerでPrivate EC2へ接続し、OSを確認。

```bash
cat /etc/os-release
```

Amazon Linux 2023へMariaDBクライアントをインストール。

```bash
sudo dnf install mariadb105
```

確認。

```bash
mysql --version
```

MariaDBクライアントを使って、RDS for MySQLへ接続できる。

---

# 16. Private EC2からRDSへ接続

Private EC2内で実行。

```bash
mysql \
  --connect-timeout=5 \
  -h RDSのEndpoint \
  -P 3306 \
  -u admin \
  -p
```

例。

```bash
mysql \
  --connect-timeout=5 \
  -h toki-rds-mysql.xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
  -P 3306 \
  -u admin \
  -p
```

意味。

```text
-h
→ 接続先ホスト名
→ RDS Endpointを指定

-P 3306
→ MySQL接続用ポート

-u admin
→ adminユーザーでログイン

-p
→ パスワード入力を求める
```

ここで使用するパスワードは、

```text
AWSコンソールのログインパスワード
IAMユーザーのパスワード
EC2のパスワード
```

ではない。

RDS作成時に設定した、

```text
MySQLのadminユーザー用パスワード
```

を入力する。

接続成功時。

```text
MariaDB [(none)]>
```

または、

```text
mysql>
```

---

# 17. Access deniedの切り分け

初回接続時に、次のエラーが表示された。

```text
Access denied for user 'admin'
```

TCP疎通確認を実行。

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/RDSのEndpoint/3306' \
  && echo "TCP 3306 OK" \
  || echo "TCP 3306 NG"
```

結果。

```text
TCP 3306 OK
```

この結果から、

```text
Private EC2
↓ TCP 3306
RDS
```

までのネットワーク経路は正常と判断した。

つまり、

```text
Security Groupの問題
Subnetの問題
Route Tableの問題
```

ではない。

`Access denied` は、

```text
RDSまで通信は届いている
↓
MySQL側の認証で拒否されている
```

という意味。

---

# 18. RDSのパスワードを再設定

CloudShellで新しいパスワードを入力。

```bash
read -s NEW_DB_PASSWORD
```

RDSのマスターパスワードを変更。

```bash
aws rds modify-db-instance \
  --db-instance-identifier toki-rds-mysql \
  --master-user-password "$NEW_DB_PASSWORD" \
  --apply-immediately
```

変数を削除。

```bash
unset NEW_DB_PASSWORD
```

状態確認。

```bash
aws rds describe-db-instances \
  --db-instance-identifier toki-rds-mysql \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text
```

`available` に戻ったあと、Private EC2から再接続。

```bash
mysql \
  --connect-timeout=5 \
  -h RDSのEndpoint \
  -P 3306 \
  -u admin \
  -p
```

新しく設定したパスワードを入力し、接続成功。

---

# 19. データベースとテーブルを作成

RDSへ接続後、MySQLプロンプト内で実行。

## 19.1 データベースを作成

```sql
CREATE DATABASE toki_test;
```

---

## 19.2 作成したデータベースを使用

```sql
USE toki_test;
```

---

## 19.3 usersテーブルを作成

```sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50)
);
```

意味。

```text
id
→ ユーザー識別番号

INT
→ 整数

PRIMARY KEY
→ 各データを一意に識別するキー

AUTO_INCREMENT
→ データ追加時に番号を自動で増やす

name
→ 名前

VARCHAR(50)
→ 最大50文字の可変長文字列
```

---

## 19.4 データを追加

```sql
INSERT INTO users (name) VALUES ('toki');
```

---

## 19.5 データを取得

```sql
SELECT * FROM users;
```

結果。

```text
+----+------+
| id | name |
+----+------+
|  1 | toki |
+----+------+
```

これにより、

```text
RDSへ接続できる
データベースを作成できる
テーブルを作成できる
データを保存できる
保存したデータを取得できる
```

ことを確認した。

---

# 20. RDS内部の確認

MySQLプロンプト内で実行。

データベース一覧。

```sql
SHOW DATABASES;
```

使用するデータベースを指定。

```sql
USE toki_test;
```

テーブル一覧。

```sql
SHOW TABLES;
```

usersテーブルの構造。

```sql
DESCRIBE users;
```

usersテーブルの中身。

```sql
SELECT * FROM users;
```

EC2はデータを保存しているわけではない。

```text
Private EC2
→ RDSへ命令を送る操作端末

RDS
→ 実際にデータを保存する場所
```

---

# 21. MySQLから抜ける

MySQLプロンプト内で実行。

```sql
exit;
```

または、

```sql
quit;
```

成功時。

```text
Bye
```

通常のEC2シェルへ戻る。

---

# 22. トラブルシューティング実践

## 22.1 RDS用Security GroupのInbound Ruleを削除

一時的に削除したルール。

```text
Type：MySQL/Aurora
Protocol：TCP
Port：3306
Source：private-sg
```

その後、Private EC2からRDSへ再接続。

```bash
mysql \
  --connect-timeout=5 \
  -h RDSのEndpoint \
  -P 3306 \
  -u admin \
  -p
```

結果。

```text
接続失敗
```

TCP疎通確認。

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/RDSのEndpoint/3306' \
  && echo "TCP 3306 OK" \
  || echo "TCP 3306 NG"
```

結果。

```text
TCP 3306 NG
```

---

## 22.2 Security Groupのルールを元に戻す

```text
Type：MySQL/Aurora
Protocol：TCP
Port：3306
Source：private-sg
```

TCP疎通確認。

```bash
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/RDSのEndpoint/3306' \
  && echo "TCP 3306 OK" \
  || echo "TCP 3306 NG"
```

結果。

```text
TCP 3306 OK
```

MySQLへ再接続。

```bash
mysql \
  --connect-timeout=5 \
  -h RDSのEndpoint \
  -P 3306 \
  -u admin \
  -p
```

接続後。

```sql
USE toki_test;
SELECT * FROM users;
```

データを再度取得できた。

---

# 23. エラーの見分け方

## TCP 3306 NG

```text
EC2からRDSへ通信できていない
```

確認する場所。

```text
RDS用SGのInbound Rule
EC2用SGのOutbound Rule
RDS Endpoint
VPC
Subnet
```

---

## TCP 3306 OK + Access denied

```text
ネットワーク経路は正常
MySQL側の認証に失敗
```

確認する場所。

```text
ユーザー名
パスワード
```

---

## dnf installが進まない

```text
Private EC2から
パッケージ保存先へ到達できない
```

今回の原因。

```text
S3 Gateway Endpointがない
```

対応。

```text
S3 Gateway Endpointを作成
↓
Private EC2用Route Tableへ関連付け
```

---

# 24. 削除手順

学習終了後、課金を避けるためにリソースを削除した。

削除順序。

```text
① RDS本体
② Private EC2
③ VPC Endpoint
④ DB Subnet Group
⑤ Security Group
⑥ Subnet
⑦ Route Table
⑧ VPC
⑨ IAM Role
```

---

## 24.1 RDS本体を削除

CloudShellで実行。

```bash
aws rds delete-db-instance \
  --db-instance-identifier toki-rds-mysql \
  --skip-final-snapshot
```

今回は学習用データのため、最終スナップショットは作成しない。

状態確認。

```bash
aws rds describe-db-instances \
  --db-instance-identifier toki-rds-mysql \
  --query "DBInstances[0].DBInstanceStatus" \
  --output text
```

削除中。

```text
deleting
```

削除完了時。

```text
DBInstanceNotFound
```

---

## 24.2 EC2を終了

```text
EC2
↓
インスタンス
↓
Private EC2を選択
↓
インスタンスを終了
```

---

## 24.3 VPC Endpointを削除

削除したEndpoint。

```text
com.amazonaws.ap-northeast-1.ssm

com.amazonaws.ap-northeast-1.ssmmessages

com.amazonaws.ap-northeast-1.ec2messages

com.amazonaws.ap-northeast-1.s3
```

---

## 24.4 DB Subnet Groupを削除

```text
RDS
↓
サブネットグループ
↓
対象のDB Subnet Group
↓
削除
```

RDS本体が残っていると削除できない。

---

## 24.5 Security Groupを削除

依存関係を考慮し、次の順番で削除。

```text
rds-sg
↓
ssm-sg
↓
private-sg
```

`rds-sg` と `ssm-sg` から `private-sg` を参照していたため、`private-sg` は最後に削除する。

---

## 24.6 Subnet・Route Table・VPCを削除

```text
Private Subnet × 2
↓
自作Route Table
↓
自作VPC
```

---

## 24.7 IAM Roleを削除

今回専用に作成したSSM用IAM Roleを削除。

---

# 25. 今回の学び

```text
Amazon RDS
→ データベースを使いやすい形で提供するマネージドサービス

DB Subnet Group
→ RDSを配置してよいSubnet候補のまとまり

異なるAZのSubnetを2つ登録
→ 別AZへ配置できる候補地を確保する

RDS用SG
→ Private EC2からのTCP 3306通信だけを許可する

RDS Endpoint
→ RDSへ接続するためのDNS名

MySQLクライアント
→ EC2からRDSへ命令を送る道具

S3 Gateway Endpoint
→ Private EC2からS3上のパッケージを取得するための経路
```

トラブルシューティング。

```text
TCP 3306 NG
→ ネットワーク経路を確認

TCP 3306 OK
Access denied
→ ユーザー名・パスワードを確認
```

今回、RDS本体だけAWS CLIで作成した。

```text
CLIを丸暗記する必要はない
↓
コンソールで選ぶ設定を
CLIのオプションで指定している
```

ことを理解できた。

---

# 26. 構築結果

以下を自力で構築・確認・削除できた。

```text
Private EC2
SSM Session Manager接続
SSM系Interface Endpoint
S3 Gateway Endpoint
DB Subnet Group
RDS用Security Group
RDS for MySQL
RDS Endpoint
MySQLクライアント
MySQL接続
データベース作成
テーブル作成
データ追加
データ取得
Security Group障害の再現
通信経路と認証エラーの切り分け
全リソース削除
```

