# AWS EC2 Web Troubleshooting Lab

## 概要

AWS EC2上にWebサーバ環境を構築し、  
ネットワーク設定・nginx・バックエンドアプリ・リバースプロキシ設定に関する障害を再現した。

各障害について、症状・ログ・確認コマンド・原因・復旧手順を整理し、  
実務で必要となる基本的な切り分け手順を記録する。

## 目的

EC2上のWebサーバ環境で発生する代表的な障害を再現し、  
Linuxコマンド・nginxログ・systemd・AWSセキュリティグループを使って原因を切り分ける力を身につける。

## 実施した障害パターン

| No | 障害内容 | 主な原因 | 確認ポイント |
|---|---|---|---|
| 01 | 外部からWebページにアクセスできない | セキュリティグループで80番ポート未許可 | Security Group, curl, nginx状態 |
| 02 | Webページが表示されない | nginx停止 | systemctl, ss, curl |
| 03 | 502 Bad Gateway | Pythonバックエンド停止 | nginx error.log, systemctl, ss |
| 04 | 404 Not Found | proxy_passのパス設定ミス | nginx設定, access.log, Python側のパス |
