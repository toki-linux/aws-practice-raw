# EC2 Web Server with S3 and EBS

## 概要

AWS上でEC2を使ったWebサーバ構成の学習記録です。

- EC2にnginxをインストールし、Webページを表示
- IAM RoleでS3にアクセス、アクセスキー不要でtest.txtを取得
- 追加EBSを作成し、/dataにマウント
- /etc/fstabで自動マウント設定
- トラブル（fstabミスでemergency mode）から復旧経験

---

## 目的

- EC2を使った基本的なWebサーバ構築を理解
- IAM RoleでアクセスキーなしにS3操作
- EBSの作成・アタッチ・マウント・自動マウント理解
- トラブル発生時の原因切り分け経験
- AWSリソース削除による課金防止

---

## 構成

