
# AWS API・移行・ハイブリッドストレージ関連サービス

## 概要

このファイルでは、以下のサービスや用語を整理する。

```text
Amazon API Gateway
API
APIリクエスト
スキーマ

AWS Migration Hub
AWS Application Migration Service
AWS Database Migration Service
AWS DataSync
AWS Snowball

AWS Storage Gateway
AWS Transfer Family

FTP
FTPS
SFTP
```

---

# APIとは？

## 概要

APIは、`Application Programming Interface` の略。

一言でいうと、以下のようなもの。

```text
API
=
システム同士がやり取りするための窓口・ルール
```

あるシステムが、別のシステムへ処理を依頼するときに使う。

---

## APIへリクエストするとは？

APIへリクエストするとは、他のシステムへ処理を依頼すること。

```text
APIリクエスト
=
他のシステムへお願いを送ること
```

---

## 天気アプリで例える

天気アプリが、大阪の天気を表示するとする。

アプリ自身が気象データを持っていない場合、気象情報を提供するサービスへ問い合わせる。

```text
天気アプリ
↓
「今日の大阪の天気を教えて」
↓
気象サービスのAPI
↓
「晴れです」
```

この問い合わせが、APIリクエスト。

---

# APIは何ができるのか？

APIは、情報を取得するだけではない。

たとえば、ショッピングアプリでは以下のような処理に使える。

```text
商品一覧を見る
ログインする
商品を注文する
注文履歴を見る
登録情報を更新する
画像をアップロードする
注文をキャンセルする
```

データ操作の基本は、以下の4つ。

```text
作成する
取得する
更新する
削除する
```

この4つは、まとめて `CRUD` と呼ばれる。

| 操作 | 英語 | 例 |
|---|---|---|
| 作成 | Create | 注文を登録する |
| 取得 | Read | 商品一覧を見る |
| 更新 | Update | 登録住所を変更する |
| 削除 | Delete | 注文を取り消す |

---

# Amazon API Gatewayとは？

## 概要

Amazon API Gatewayは、APIを作成・公開・管理するためのAWSサービス。

```text
Amazon API Gateway
=
APIの入口をAWS上に作り、
公開・管理するサービス
```

---

## 基本構成

代表的な構成は以下。

```text
ユーザー・スマホアプリ
↓
API Gateway
↓
Lambda
↓
DynamoDB
```

API Gatewayがリクエストを受け取り、Lambdaなどの処理担当へ渡す。

Lambdaが処理を実行し、必要に応じてDynamoDBなどへアクセスする。

---

## API Gatewayがすること

API Gatewayには、以下のような役割がある。

```text
リクエストを受け取る
接続先へリクエストを渡す
パスやHTTPメソッドに応じて処理を振り分ける
認証・認可を行う
アクセス数を制限する
ログを残す
レスポンスを返す
```

---

## API Gatewayがしないこと

API Gateway自身が、注文処理や商品検索の本体を実行するわけではない。

```text
API Gateway
→ 受付

Lambda・EC2・ECSなど
→ 実際の処理担当

DynamoDB・RDSなど
→ データの保存場所
```

---

# API GatewayとAPIの違い

## API

```text
API
=
どのようなお願いを送れるか
どの形式で送るか
どのような返事が返るか
というルール
```

例として、以下のようなAPIを考える。

```text
GET /users/1
→ IDが1のユーザー情報を取得する

POST /orders
→ 新しい注文を登録する
```

---

## API Gateway

```text
API Gateway
=
APIの入口をAWS上に作り、
受付・振り分け・管理を行うサービス
```

---

## ドライブスルーで例える

```text
API
→ 注文メニュー・注文方法・受け答えのルール

APIリクエスト
→ 「ハンバーガーを1つください」という注文

API Gateway
→ 注文を受け付ける窓口

Lambda
→ 厨房で実際に調理する担当

DynamoDB
→ 注文履歴などを保存する保管庫
```

全体の流れは以下。

```text
お客さん
↓ 注文
API Gateway：注文窓口
↓
Lambda：厨房
↓
DynamoDB：注文記録の保管庫
↓
Lambda：処理結果を返す
↓
API Gateway：返事を渡す
↓
お客さん
```

---

# API GatewayとLambdaの組み合わせ

## 例：ユーザー情報を取得する

スマホアプリから、以下のリクエストを送る。

```text
GET /users/1
```

全体の流れは以下。

```text
スマホアプリ
↓ GET /users/1
API Gateway
↓
Lambda
↓
DynamoDB
↓
Lambda
↓
API Gateway
↓
スマホアプリ
```

DynamoDBに以下のデータがあるとする。

| user_id | name | age |
|---:|---|---:|
| 1 | toki | 21 |

返されるデータの例。

```json
{
  "user_id": 1,
  "name": "toki",
  "age": 21
}
```

---

# EC2・Lambda・DynamoDBの使い分け

## Amazon EC2

EC2は、仮想サーバ。

OSやミドルウェアを自分で管理したい場合に向く。

```text
EC2が向く場面

常に動かすアプリケーション
OSを自由に操作したい
nginxやsystemdを自分で管理したい
細かい設定を自分で決めたい
```

---

## AWS Lambda

Lambdaは、必要なときだけコードを実行するサービス。

サーバそのものを自分で管理する必要がない。

```text
Lambdaが向く場面

短時間で終わる処理
イベントが起きたときだけ動かしたい
APIリクエストが来たときだけ処理したい
サーバ管理を減らしたい
```

---

## Amazon DynamoDB

DynamoDBは、高速なNoSQLデータベース。

キーを使ってデータを素早く読み書きしたい場合に向く。

```text
DynamoDBが向く場面

キーで高速にデータを取得したい
大量アクセスへ対応したい
自動でスケールしてほしい
データベースサーバを管理したくない
```

---

## よくあるサーバレス構成

```text
ユーザー
↓
API Gateway
↓
Lambda
↓
DynamoDB
```

この構成では、EC2のOS管理や常時稼働を意識する必要が少ない。

---

# スキーマとは？

## 概要

スキーマは、データ構造の設計図。

```text
スキーマ
=
データの設計図
```

---

## テーブルで例える

ユーザーテーブルを作るとする。

| 項目 | データ型 | 意味 |
|---|---|---|
| user_id | 整数 | ユーザーID |
| name | 文字列 | 名前 |
| age | 整数 | 年齢 |

実際のデータは以下。

| user_id | name | age |
|---:|---|---:|
| 1 | toki | 21 |
| 2 | sachi | 30 |

```text
スキーマ
→ どの項目があるか
→ どのデータ型を使うか
→ どのような制約を付けるか

データ
→ 実際に保存されている値
```

---

## RDSとDynamoDBの違い

```text
RDS
→ スキーマを比較的厳密に決める

DynamoDB
→ データ構造を柔軟に扱いやすい
```

ただし、DynamoDBでもキーなどの基本設計は必要。

---

# AWSへの移行とは？

## 概要

会社内のサーバやデータを、AWSへ移すことを考える。

```text
オンプレミス環境
↓
AWSへ移行
↓
クラウド上で運用
```

移行対象によって、使うサービスが異なる。

---

# AWS Migration Hubとは？

## 概要

AWS Migration Hubは、AWSへの移行作業全体を追跡・管理するサービス。

```text
AWS Migration Hub
=
移行作業全体の進捗をまとめて確認する管理画面
```

---

## 何をするのか？

サーバやアプリをAWSへ移す場合、移行作業は複数に分かれる。

```text
移行対象を調査する
対象を整理する
移行ツールを実行する
進捗を確認する
問題を確認する
移行後の状態を確認する
```

Migration Hubは、複数の移行作業の状況をまとめて見えるようにする。

---

## 引っ越しで例える

```text
Migration Hub
→ 引っ越し全体の進捗表

Application Migration Serviceなど
→ 実際に荷物を運ぶトラック
```

Migration Hub自身が、サーバやデータを直接運ぶわけではない。

---

# 移行系サービスの整理

| サービス | 主な役割 |
|---|---|
| AWS Migration Hub | 移行全体の進捗を管理する |
| AWS Application Migration Service | サーバをAWSへ移行する |
| AWS Database Migration Service | データベースを移行する |
| AWS DataSync | ファイルなどのデータを転送する |
| AWS Snowball | 大量データを物理デバイスで移行する |

---

## AWS Application Migration Service

AWS Application Migration Serviceは、サーバをAWSへ移行するためのサービス。

```text
既存サーバ
↓
Application Migration Service
↓
AWS上の環境へ移行
```

覚え方は以下。

```text
Application Migration Service
→ サーバ移行
```

---

## AWS Database Migration Service

AWS Database Migration Serviceは、データベースを移行するためのサービス。

略して、`AWS DMS` と呼ばれる。

```text
既存データベース
↓
AWS DMS
↓
AWS上のデータベース
```

覚え方は以下。

```text
Database Migration Service
→ データベース移行
```

---

## AWS DataSync

AWS DataSyncは、オンプレミス環境やAWSサービス間で、ファイルなどのデータを転送するためのサービス。

```text
オンプレミスのファイル
↓
AWS DataSync
↓
AWSストレージ
```

覚え方は以下。

```text
DataSync
→ ファイルなどのデータ転送
```

---

## AWS Snowball

AWS Snowballは、大量データを物理デバイスへ保存し、AWSへ送ることでデータを移行するサービス。

```text
大量データ
↓
Snowballデバイスへ保存
↓
物理的に配送
↓
AWSへ取り込む
```

ネットワーク転送では時間がかかりすぎる場合に使う。

---

# AWS Storage Gatewayとは？

## 概要

AWS Storage Gatewayは、オンプレミス環境とAWSストレージを接続するハイブリッドクラウドサービス。

```text
AWS Storage Gateway
=
オンプレミス環境とAWSストレージをつなぐ橋
```

---

## 基本構成

```text
会社内のサーバ
↓
Storage Gateway
↓
AWSストレージ
```

会社側のシステムから見ると、既存のファイルサーバやディスクのように使える。

裏側では、AWSのストレージへデータが保存される。

---

## 倉庫で例える

```text
会社の倉庫
↓
近くの受付窓口
↓
AWSの巨大倉庫
```

Storage Gatewayは、この受付窓口に近い。

会社側のシステムは、近くの窓口へ荷物を預ける感覚で使う。

実際の保管先は、AWS側にある。

---

# Storage Gatewayの代表的な種類

| 種類 | 主な役割 |
|---|---|
| File Gateway | ファイルとしてアクセスし、AWSストレージへ保存する |
| Volume Gateway | ブロックストレージとして利用し、AWSへ保存・バックアップする |
| Tape Gateway | 物理テープの代わりに、仮想テープとしてAWSへ保存する |

---

## File Gateway

File Gatewayは、オンプレミス側からファイルとして使える。

```text
オンプレミスのサーバ
↓ ファイルとして保存
File Gateway
↓
S3など
```

---

## Volume Gateway

Volume Gatewayは、ディスクのようなブロックストレージとして使える。

```text
オンプレミスのサーバ
↓ ディスクとして利用
Volume Gateway
↓
AWSへ保存・バックアップ
```

---

## Tape Gateway

Tape Gatewayは、物理テープバックアップの代わりに使える。

```text
既存のバックアップソフト
↓
Tape Gateway
↓
仮想テープとしてAWSへ保存
```

---

# AWS Transfer Familyとは？

## 概要

AWS Transfer Familyは、一般的なファイル転送プロトコルを使い、AWSストレージへファイルを送受信するためのサービス。

```text
AWS Transfer Family
=
既存のファイル転送方式を使って、
S3やEFSなどへファイルを送受信するサービス
```

---

## 基本構成

```text
外部の取引先・社内システム
↓
SFTP・FTPS・FTPなど
↓
AWS Transfer Family
↓
S3・EFSなど
```

---

## どのような場面で使うのか？

既存システムでは、以下のような運用が残っていることがある。

```text
FTPでファイルを送る
SFTPで請求データを送る
毎日CSVファイルをアップロードする
取引先からファイルを受け取る
```

Transfer Familyを使うと、送信側の使い方を大きく変えずに、保存先をAWSへ移行しやすい。

---

## 倉庫で例える

```text
Transfer Family
→ ファイル受け取り窓口

S3・EFS
→ ファイルの保管倉庫
```

取引先は、今までどおりSFTPなどでファイルを送る。

裏側では、AWSのストレージへ保存される。

---

# Storage GatewayとTransfer Familyの違い

| サービス | 主な役割 |
|---|---|
| Storage Gateway | オンプレミス側から既存ストレージのようにAWSストレージを使う |
| Transfer Family | SFTP・FTPS・FTPなどでファイルを送受信する入口を作る |

```text
Storage Gateway
→ オンプレミスのストレージとAWSをつなぐ橋

Transfer Family
→ ファイル転送プロトコルで受け付ける窓口
```

---

# FTP・FTPS・SFTPとは？

## 概要

FTP・FTPS・SFTPは、すべてファイル転送に使うプロトコル。

```text
ファイル転送プロトコル
=
PCやサーバ間でファイルを送受信するためのルール
```

---

## FTP

FTPは、`File Transfer Protocol` の略。

```text
FTP
→ 昔から使われているファイル転送プロトコル
→ 通信内容が暗号化されない
```

---

## FTPS

FTPSは、FTPへSSL/TLSによる暗号化を加えた方式。

```text
FTPS
→ FTPをベースにする
→ SSL/TLSで通信を暗号化する
```

---

## SFTP

SFTPは、SSHを利用するファイル転送方式。

```text
SFTP
→ SSHをベースにする
→ 通信を暗号化する
```

---

## 違い

| プロトコル | 暗号化 | ベースとなる仕組み |
|---|---|---|
| FTP | なし | FTP |
| FTPS | あり | FTP + SSL/TLS |
| SFTP | あり | SSH |

---

# 全体のまとめ

```text
API
→ システム同士がやり取りするための窓口・ルール

APIリクエスト
→ 他のシステムへ処理を依頼すること

API Gateway
→ APIの入口を作成・公開・管理するサービス

API Gateway
→ 受付担当

Lambda・EC2・ECSなど
→ 実際の処理担当

DynamoDB・RDSなど
→ データの保存場所

スキーマ
→ データ構造の設計図

Migration Hub
→ 移行全体の進捗管理

Application Migration Service
→ サーバ移行

Database Migration Service
→ データベース移行

DataSync
→ ファイルなどのデータ転送

Snowball
→ 物理デバイスを使った大量データ移行

Storage Gateway
→ オンプレミスとAWSストレージをつなぐ橋

Transfer Family
→ ファイル転送プロトコルでS3やEFSなどへ送受信する入口

FTP
→ 暗号化なし

FTPS
→ FTP + SSL/TLS

SFTP
→ SSHを利用するファイル転送
```


