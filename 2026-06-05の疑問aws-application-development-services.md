
# AWSのアプリケーション開発・デプロイ・認証サービス

## 概要

AWSには、Webアプリケーションやモバイルアプリを開発・公開・運用するためのサービスがある。

大きく分けると、以下のように整理できる。

```text
Amplify
→ Web・モバイルアプリ開発を簡単にする

Codeシリーズ
→ コードの保存、ビルド、テスト、デプロイを支援する

Cognito
→ アプリのユーザー登録・ログイン・権限付与を担当する
```

---

# AWS Amplifyとは？

## 概要

AWS Amplifyは、WebアプリケーションやモバイルアプリをAWS上で作りやすくするためのサービス群。

```text
AWS Amplify
=
Web・モバイルアプリの構築、接続、公開を
簡単にするための便利セット
```

---

## Amplifyが支援するもの

たとえば、アプリに以下の機能を追加したいとする。

```text
ログイン機能
データ保存
画像アップロード
API呼び出し
Webサイトの公開
```

通常は、複数のAWSサービスを自分で組み合わせる必要がある。

```text
Cognito
S3
DynamoDB
AppSync
Lambda
IAM
CloudFront
```

Amplifyを利用すると、アプリ開発者がこれらを扱いやすくなる。

---

# Amplifyの主な要素

Amplifyを学ぶとき、以下の名前が出てくることがある。

```text
Amplify Hosting
Amplify Libraries
Amplify CLI
Amplify Studio
```

---

## Amplify Hosting

Amplify Hostingは、WebサイトやWebアプリを公開する機能。

```text
React・Next.jsなどで作成したアプリ
↓
Amplify Hosting
↓
インターネットへ公開
```

コードの変更に応じて、ビルドやデプロイを行う仕組みも利用できる。

```text
Amplify Hosting
=
Webアプリを公開する機能
```

---

## Amplify Libraries

Amplify Librariesは、アプリケーションのコードからAWSの機能を使いやすくするためのライブラリ。

たとえば、以下の機能をコードから扱いやすくする。

```text
ユーザー登録
ログイン
データ取得
ファイルアップロード
API呼び出し
```

イメージは以下。

```text
アプリケーション
↓
Amplify Libraries
↓
Cognito・S3・APIなど
```

---

## Amplify CLI

Amplify CLIは、ターミナルからAmplify関連の環境を操作するための道具。

古いAmplify Gen 1の教材では、以下のようなコマンドがよく登場する。

```bash
amplify init
amplify add auth
amplify push
```

イメージは以下。

```text
amplify init
→ Amplify環境を初期設定する

amplify add auth
→ ログイン機能を追加する

amplify push
→ AWS上へ設定を反映する
```

---

## Amplify Studio

Amplify Studioは、画面上でアプリ開発やバックエンド設定を支援するツールとして紹介されることがある。

```text
Amplify Studio
=
GUIでアプリ開発や設定を支援するツール
```

---

## 現在のAmplify Gen 2

現在の公式ドキュメントでは、Amplify Gen 2が中心。

Gen 2では、TypeScriptを使ってバックエンドを定義する方法が前面に出ている。

```text
古い教材
→ Gen 1
→ CLIコマンド中心の説明が多い

現在の公式ドキュメント
→ Gen 2
→ TypeScriptを使った定義が中心
```

Cloud Practitionerの試験対策では、まず以下を押さえる。

```text
Amplify
→ Web・モバイルアプリ開発を簡単にするサービス群

Hosting
→ Webアプリを公開する

Libraries
→ コードからAWS機能を使いやすくする

CLI
→ コマンドからAmplify環境を操作する

Studio
→ GUIで開発や設定を支援する
```

---

# アプリが公開されるまでの流れ

## 全体像

Amplifyを使ってアプリを作成する場合、概念としては以下のように考える。

```text
1. アプリ本体を作る
↓
2. 必要なAWS機能を追加する
↓
3. AWS上へ反映する
↓
4. アプリのコードからAWS機能を利用する
↓
5. アプリを公開する
↓
6. ユーザーが利用する
```

---

## 具体例

### 1. アプリ本体を作る

```text
React
Next.js
Flutter
Swift
```

などを使ってアプリを作る。

### 2. 必要なAWS機能を追加する

```text
ログイン
→ Cognito

データ保存
→ DynamoDBなど

画像保存
→ S3

独自処理
→ Lambda
```

### 3. AWS上へ反映する

設定したバックエンドをAWS上へ作成する。

### 4. アプリからAWS機能を使う

```text
アプリ
↓
Amplify Libraries
↓
Cognito・S3・APIなど
```

### 5. アプリを公開する

```text
アプリ
↓
Amplify Hosting
↓
インターネットへ公開
```

---

# 開発・デプロイ系のCodeシリーズ

## 概要

AWSには、コードの保存、ビルド、テスト、デプロイを支援するサービスがある。

代表例は以下。

```text
CodeCommit
CodeBuild
CodeDeploy
CodePipeline
CodeArtifact
```

---

## 全体の流れ

```text
コードを書く
↓
CodeCommitなどへ保存
↓
CodeBuildでビルド・テスト
↓
CodeDeployで環境へ配置
↓
CodePipelineで一連の流れを自動化
```

CodeArtifactは、再利用するライブラリやパッケージを保存するために使う。

---

# AWS CodeCommitとは？

## 概要

AWS CodeCommitは、GitリポジトリをAWS上で管理するサービス。

```text
CodeCommit
=
ソースコードの保管場所
=
Gitリポジトリ
```

---

## GitHubとの関係

役割のイメージは、GitHubのリポジトリに近い。

```text
開発者
↓
git push
↓
CodeCommit
↓
ソースコードを保存
```

---

## 現在の状況

CodeCommitは、2024年に新規顧客の受付を一度停止した。

その後、2025年11月から再び新規顧客も利用可能になった。

```text
2024年
→ 新規顧客の受付を停止

2025年11月
→ 新規顧客の受付を再開
```

---

# AWS CodeBuildとは？

## 概要

AWS CodeBuildは、コードをビルド・テストするサービス。

```text
CodeBuild
=
コードをビルド・テストするサービス
```

---

## ビルドとは？

ビルドは、ソースコードを実行・配布できる形へ変換する処理。

```text
ソースコード
↓
ビルド
↓
実行・配布しやすい形
```

---

## CodeBuildの役割

```text
ソースコードを取得する
↓
必要な依存関係を取得する
↓
テストを実行する
↓
ビルドする
↓
成果物を出力する
```

---

# AWS CodeDeployとは？

## 概要

AWS CodeDeployは、アプリケーションを実行環境へ配置する作業を自動化するサービス。

```text
CodeDeploy
=
アプリケーションの配置を自動化するサービス
```

---

## デプロイとは？

デプロイは、作成したアプリケーションを利用できる環境へ配置・反映すること。

```text
完成したアプリ
↓
サーバなどへ配置
↓
ユーザーが利用できる状態にする
```

---

## CodeDeployの配置先の例

```text
EC2
Lambda
ECS
```

---

# AWS CodePipelineとは？

## 概要

AWS CodePipelineは、ソフトウェアをリリースするまでの流れを自動化するサービス。

```text
CodePipeline
=
コード変更からデプロイまでの流れを
自動化するサービス
```

---

## イメージ

```text
ソースコードを更新
↓
CodePipelineが変更を検知
↓
CodeBuildでテスト・ビルド
↓
CodeDeployで配置
↓
新しいアプリを公開
```

---

## CI/CDとは？

CodePipelineを学ぶと、`CI/CD` という言葉がよく出てくる。

```text
CI
=
Continuous Integration
=
継続的インテグレーション

CD
=
Continuous Delivery
または
Continuous Deployment
=
継続的デリバリー・継続的デプロイ
```

初心者向けには、以下のように理解する。

```text
コードを変更
↓
自動でテスト
↓
自動でビルド
↓
自動でデプロイ
```

---

# AWS CodeArtifactとは？

## 概要

AWS CodeArtifactは、ソフトウェアパッケージを保管・共有するためのサービス。

```text
CodeArtifact
=
ライブラリやパッケージの保管場所
```

---

## 何を保存するのか？

アプリケーション開発では、外部のライブラリや社内共通部品を使うことがある。

```text
npmパッケージ
Pythonパッケージ
Javaのパッケージ
.NETのパッケージ
```

CodeArtifactは、これらを安全に保存・共有する。

---

# Codeシリーズを例えで整理する

本を出版する流れに例える。

```text
CodeCommit
→ 原稿を保存する場所

CodeBuild
→ 原稿のチェック・製本をする場所

CodeDeploy
→ 完成した本を本屋へ並べる係

CodePipeline
→ 原稿提出から販売までを自動化するベルトコンベア

CodeArtifact
→ よく使う素材や部品を保管する倉庫
```

---

## 一覧表

| サービス | 役割 |
|---|---|
| CodeCommit | ソースコードを保存するGitリポジトリ |
| CodeBuild | コードをビルド・テストする |
| CodeDeploy | アプリケーションの配置を自動化する |
| CodePipeline | リリースまでの流れを自動化する |
| CodeArtifact | ライブラリやパッケージを保管・共有する |

---

# Amazon Cognitoとは？

## 概要

Amazon Cognitoは、Webアプリやモバイルアプリの認証・ユーザー管理を支援するサービス。

```text
Amazon Cognito
=
アプリのログイン機能を作るサービス
```

---

## Cognitoでできること

```text
ユーザー登録
ログイン
パスワード管理
認証
SNSログイン
一時的なAWS権限の付与
```

---

# Cognitoユーザープールとは？

## 概要

Cognitoユーザープールは、アプリのユーザー名簿。

```text
Cognito User Pool
=
アプリのユーザー名簿
=
ユーザー登録・ログインを管理する
```

---

## 管理する情報の例

```text
メールアドレス
ユーザー名
パスワード
電話番号
ユーザー属性
ログイン状態
```

---

## ユーザープールの役割

```text
アプリ
↓
ユーザープール
↓
「この人は登録済みか？」
↓
「正しいパスワードか？」
↓
ログイン成功
```

一言で表すと以下。

```text
ユーザープール
→ あなたは誰ですか？
```

---

# Cognito IDプールとは？

## 概要

Cognito IDプールは、アプリのユーザーへ一時的なAWS認証情報を渡す仕組み。

```text
Cognito Identity Pool
=
ユーザーへ一時的なAWS認証情報を渡す
```

---

## 何のために使うのか？

たとえば、ログイン済みユーザーに、自分の画像をS3へアップロードさせたいとする。

```text
ログインしたユーザー
↓
IDプール
↓
一時的なAWS認証情報
↓
許可された範囲でS3へアクセス
```

---

## IDプールの役割

一言で表すと以下。

```text
IDプール
→ あなたは何ができますか？
```

---

# ユーザープールとIDプールの違い

| 種類 | 主な役割 | 覚え方 |
|---|---|---|
| Cognito User Pool | ユーザー登録・ログイン・本人確認 | あなたは誰ですか？ |
| Cognito Identity Pool | 一時的なAWS認証情報を付与 | あなたは何ができますか？ |

---

## 受付で例える

```text
ユーザープール
→ 受付で本人確認をする

IDプール
→ 入館後に権限付きの入館カードを渡す
```

---

## 全体の流れ

```text
ユーザー
↓
ユーザープールでログイン
↓
本人確認に成功
↓
IDプール
↓
一時的なAWS認証情報を受け取る
↓
許可された範囲でS3などへアクセス
```

---

## 注意点

ユーザープールとIDプールは、必ずセットで使うわけではない。

```text
ログイン機能だけ必要
→ ユーザープールを使う

AWSリソースへ一時的な権限を渡したい
→ IDプールを使う
```

---

# AmplifyとCognitoの関係

Amplifyからログイン機能を追加すると、裏側でCognitoを利用する構成になることがある。

```text
アプリ開発者
↓
Amplifyで認証機能を追加
↓
裏側でCognitoを利用
↓
ユーザー登録・ログイン機能を提供
```

---

# 全体のまとめ

```text
Amplify
→ Web・モバイルアプリ開発を簡単にするサービス群

Amplify Hosting
→ Webアプリを公開する

Amplify Libraries
→ アプリコードからAWS機能を使いやすくする

Amplify CLI
→ コマンドからAmplify環境を操作する

Amplify Studio
→ GUIでアプリ開発や設定を支援する

CodeCommit
→ ソースコードを保存するGitリポジトリ

CodeBuild
→ コードをビルド・テストする

CodeDeploy
→ アプリケーションの配置を自動化する

CodePipeline
→ リリースまでの一連の流れを自動化する

CodeArtifact
→ ライブラリやパッケージを保管・共有する

Amazon Cognito
→ アプリの認証・ユーザー管理サービス

Cognito User Pool
→ ユーザー登録・ログインを管理する
→ あなたは誰ですか？

Cognito Identity Pool
→ 一時的なAWS認証情報を渡す
→ あなたは何ができますか？
```

