
# AWSの操作方法・設定形式・構築支援サービス

## 概要

AWSのサービスは、さまざまな方法で操作できる。

代表例は以下。

```text
AWS Management Console
→ 人間がブラウザ画面から操作する

AWS CLI
→ 人間がターミナルからコマンドで操作する

AWS SDK
→ プログラムがコードから操作する

AWS API
→ AWSサービスへ処理を依頼する窓口
```

---

# APIとは？

## 概要

APIは、`Application Programming Interface` の略。

一言でいうと、以下のようなもの。

```text
API
=
システム同士が会話するための窓口
```

---

## APIへリクエストするとは？

APIへリクエストするとは、他のシステムへ処理を依頼すること。

```text
APIリクエスト
=
他のシステムへお願いをすること
```

---

## 天気アプリで例える

天気アプリを作るとする。

天気アプリ自身が、気象データを持っているとは限らない。

そこで、気象情報を提供するシステムへ問い合わせる。

```text
天気アプリ
↓
「今日の大阪の天気を教えて」
↓
気象サービスのAPI
↓
「晴れです」
```

これがAPIリクエスト。

---

## AWSサービスもAPIで動く

AWSサービスの操作も、裏側ではAPIリクエストとして処理される。

たとえば、S3バケットを作成するとする。

```text
S3バケットを作成してほしい
↓
S3 APIへリクエスト
↓
S3バケットが作成される
```

---

# AWS Management Consoleとは？

## 概要

AWS Management Consoleは、ブラウザから利用するAWSの管理画面。

```text
AWS Management Console
=
ブラウザから見るAWSの管理画面
```

普段、AWSのWeb画面でEC2やS3などを操作している場合、Management Consoleを利用している。

---

## 例

```text
ブラウザ
↓
AWS Management Console
↓
EC2インスタンスを作成
↓
裏側でEC2 APIへリクエスト
```

画面から操作していても、裏側ではAPIが使われている。

---

# AWS CLIとは？

## 概要

AWS CLIは、ターミナルからコマンドでAWSを操作するための道具。

```text
AWS CLI
=
人間がコマンドでAWSへお願いする道具
```

---

## 例

S3バケットの一覧を確認する。

```bash
aws s3 ls
```

EC2インスタンスの情報を確認する。

```bash
aws ec2 describe-instances
```

---

# SDKとは？

## 概要

SDKは、`Software Development Kit` の略。

一言でいうと、以下のようなもの。

```text
SDK
=
プログラムからAWSを操作しやすくするための道具セット
```

---

## SDKを使う流れ

PythonプログラムからS3へアクセスする例。

```text
Pythonプログラム
↓
AWS SDK
↓
S3 APIへリクエスト
↓
S3バケット一覧を取得
```

---

## Pythonの例

Pythonでは、AWS SDKとして `boto3` がよく使われる。

```python
import boto3

s3 = boto3.client("s3")
response = s3.list_buckets()

for bucket in response["Buckets"]:
    print(bucket["Name"])
```

このコードは、S3 APIへリクエストして、バケット一覧を取得する。

---

## CLIとSDKの違い

```text
AWS CLI
→ 人間がコマンドでAWSへお願いする

AWS SDK
→ プログラムがコードでAWSへお願いする
```

---

# サードパーティーデータとは？

## 概要

サードパーティーデータは、自社や自分たちが直接集めたものではなく、外部の第三者から提供されるデータ。

```text
サードパーティーデータ
=
外部の第三者が提供するデータ
```

---

## 例

```text
天気情報会社が提供する気象データ
地図会社が提供する位置情報
市場調査会社が提供する統計データ
広告会社が提供する分析データ
```

---

## APIとの関係

外部サービスのAPIを使い、サードパーティーデータを取得することがある。

```text
自社アプリ
↓
外部企業のAPIへリクエスト
↓
サードパーティーデータを取得
```

---

# JSON形式とYAML形式とは？

## 概要

JSONとYAMLは、データや設定情報を書くための形式。

コードのように見えるが、正確にはデータの書き方・設定の書き方。

```text
JSON・YAML
=
人間とコンピュータが読める設定メモ
```

---

## JSON形式

JSONでは、`{ }` や `" "` を使ってデータを表現する。

```json
{
  "name": "toki",
  "age": 21,
  "service": "EC2"
}
```

---

## YAML形式

YAMLでは、主にインデントを使って構造を表現する。

```yaml
name: toki
age: 21
service: EC2
```

---

## 違い

```text
JSON
→ { } や " " が多い
→ コンピュータが扱いやすい

YAML
→ インデントで構造を表す
→ 人間が読みやすい
```

---

## AWSではどこで使うのか？

たとえば、CloudFormationのテンプレートで使われる。

```text
CloudFormation
↓
JSONまたはYAMLで構成を書く
↓
AWSリソースを作成する
```

---

# AWS CloudFormationとは？

## 概要

AWS CloudFormationは、AWSリソースをテンプレートから作成・管理するサービス。

```text
CloudFormation
=
AWS環境を設計図から作るサービス
```

---

## イメージ

YAMLまたはJSONで、作成したいAWSリソースを書く。

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
```

CloudFormationがテンプレートを読み込み、S3バケットを作成する。

```text
YAML・JSONテンプレート
↓
CloudFormation
↓
AWSリソースを作成
```

---

## 何を作れるのか？

CloudFormationでは、さまざまなAWSリソースを作成できる。

```text
VPC
Subnet
EC2
Security Group
S3
IAM Role
RDS
```

---

# AWS Elastic Beanstalkとは？

## 概要

AWS Elastic Beanstalkは、Webアプリケーションの実行環境を簡単に構築・運用するためのサービス。

```text
Elastic Beanstalk
=
Webアプリケーションの実行環境を
いい感じに用意してくれるサービス
```

---

## 何をしてくれるのか？

アプリケーションのコードを用意すると、Elastic Beanstalkが実行環境を構築する。

代表例は以下。

```text
EC2インスタンスの用意
ロードバランサーの設定
Auto Scaling
ヘルスチェック
アプリケーションのデプロイ
```

---

# CloudFormationとElastic Beanstalkの違い

## 共通点

どちらもAWS上の環境構築を支援する。

```text
CloudFormation
→ テンプレートからAWSリソースを作成する

Elastic Beanstalk
→ 設定に基づいてWebアプリ実行環境を作成する
```

---

## 違い

違いは、主に目的と自由度。

| サービス | 目的 | 自由度 |
|---|---|---|
| CloudFormation | AWSリソース全般を設計図どおりに作る | 高い |
| Elastic Beanstalk | Webアプリの実行環境を簡単に作る | CloudFormationよりお任せが多い |

---

## 家づくりで例える

### CloudFormation

```text
家の設計図を書く

部屋
配線
玄関
窓
設備

細かく指定できる
```

### Elastic Beanstalk

```text
住宅メーカーのWebアプリ用標準プラン

Webアプリを動かしたい
↓
必要な環境をまとめて用意してくれる
```

---

## 覚え方

```text
CloudFormation
→ 何をどう作るか細かく書く

Elastic Beanstalk
→ Webアプリを動かしたい
→ 細かい裏側はある程度任せる
```

---

# AWS Device Farmとは？

## 概要

AWS Device Farmは、実際のスマートフォンやタブレットなどを使って、アプリケーションをテストするためのサービス。

```text
AWS Device Farm
=
AWSが用意した実機で
Web・モバイルアプリをテストするサービス
```

---

## なぜ必要なのか？

アプリケーションを作成したとする。

さまざまな端末で正常に動くか確認したい。

```text
iPhone
Androidスマートフォン
タブレット
複数のWebブラウザ
```

しかし、すべての端末を自分で購入・管理するのは大変。

---

## Device Farmを使う場合

```text
開発者
↓
アプリをアップロード
↓
AWS Device Farm
↓
実際の端末でテスト
↓
テスト結果を確認
```

---

## Device Farmで確認できること

```text
アプリが起動するか
画面が崩れないか
ボタンを押せるか
途中でアプリが落ちないか
複数端末で正常に動くか
```

---

## 名前で覚える

```text
Device
→ 端末

Farm
→ たくさん集めた場所

Device Farm
→ 端末がたくさん集まった場所
```

---

# Amazon WorkSpacesとは？

## 概要

Amazon WorkSpacesは、AWS上に仮想デスクトップ環境を作るサービス。

```text
Amazon WorkSpaces
=
AWS上に作る仮想デスクトップ
```

---

## イメージ

```text
自分のPC
↓ インターネット
AWS上の仮想デスクトップ
```

自分のPCは、画面・キーボード・マウスとして利用する。

実際に処理を行うPC本体は、AWS側にあるイメージ。

---

## 何に使うのか？

会社で業務用PC環境を提供する場面などで使われる。

```text
社員へ業務用デスクトップ環境を配る
在宅勤務でも会社用の環境を使えるようにする
端末内へ業務データを残しにくくする
必要に応じてユーザーを追加・削除する
```

---

# VirtualBoxとの違い

## VirtualBox

VirtualBoxは、自分のPCの中へ仮想マシンを作るソフトウェア。

```text
Mac本体
└── Ubuntu仮想マシン
```

自分のMacのCPU、メモリ、ストレージを使う。

---

## Amazon WorkSpaces

WorkSpacesは、AWS上へ仮想デスクトップを作る。

```text
AWS
└── Windows・Linuxの仮想デスクトップ
```

自分のPCは、AWS側のデスクトップへ接続する端末として使う。

---

## 比較

| 項目 | VirtualBox | Amazon WorkSpaces |
|---|---|---|
| 仮想PCを置く場所 | 自分のPC内 | AWS上 |
| CPU・メモリ | 自分のPCを使う | AWS側のリソースを使う |
| 主な用途 | 学習、検証、ローカル開発 | 業務用仮想デスクトップ |
| 接続 | 自分のPC上で起動 | ネットワーク経由で接続 |

---

# 全体のまとめ

```text
API
→ システム同士が会話するための窓口

APIリクエスト
→ 他のシステムへ処理を依頼すること

AWS Management Console
→ ブラウザから利用するAWSの管理画面

AWS CLI
→ 人間がコマンドでAWSを操作する道具

AWS SDK
→ プログラムからAWSを操作するための道具セット

サードパーティーデータ
→ 外部の第三者が提供するデータ

JSON
→ { } や " " を使うデータ・設定形式

YAML
→ インデントを使うデータ・設定形式

CloudFormation
→ AWS環境を設計図から作るサービス

Elastic Beanstalk
→ Webアプリ実行環境を簡単に作るサービス

Device Farm
→ AWSが用意した実機でアプリをテストするサービス

Amazon WorkSpaces
→ AWS上に作る仮想デスクトップ

VirtualBox
→ 自分のPC内に仮想マシンを作るソフトウェア
```

