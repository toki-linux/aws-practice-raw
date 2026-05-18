# AWS EC2 Web Server Lab

## 概要

AWS EC2上にUbuntuサーバを作成し、SSH接続後にnginxを導入して、ブラウザからWebページを表示するまでの流れを実践した。

このリポジトリでは、EC2インスタンス作成、セキュリティグループ設定、SSH接続、nginxによるWebページ公開までの手順をまとめる。

## 目的

- AWS上に仮想サーバを作成する流れを理解する
- キーペアを使ったSSH接続を実践する
- セキュリティグループで通信制御を行う
- nginxを使ってWebページを公開する
- 作成したAWSリソースを削除し、不要な課金を防ぐ流れを確認する

## 構成

```text
aws-ec2-web-server-lab/
├── README.md
├── docs/
│   ├── ec2_setup.md
│   ├── ssh_connection.md
│   ├── security_group.md
│   ├── nginx_setup.md
```

## 使用した環境

- AWS EC2
- Ubuntu Server
- nginx
- macOS Terminal
- SSH
- IAMユーザー：toki-admin

## 実施内容

1. EC2インスタンスを作成
2. キーペアを作成
3. セキュリティグループでSSHとHTTPを許可
4. SSHでEC2に接続
5. nginxをインストール
6. `/var/www/html/index.html` を作成
7. ブラウザからパブリックIPv4アドレスにアクセス
8. Webページが表示されることを確認
9. 不要なリソースを削除

## 作成したWebページ

```html
<h1>Hello from EC2</h1>
<p>My first AWS web server.</p>
```

## 確認コマンド

```bash
whoami
hostname
systemctl status nginx
curl http://localhost
```

## 学んだこと

- EC2はAWS上に作成する仮想サーバである
- AWS管理画面にログインするIAMユーザーと、EC2内で操作するLinuxユーザーは別物である
- UbuntuのEC2では、SSH接続時のユーザー名は基本的に `ubuntu` を使う
- キーペアの秘密鍵はSSH接続に必要であり、権限を適切に設定する必要がある
- セキュリティグループで22番ポートを開けないとSSH接続できない
- セキュリティグループで80番ポートを開けないとブラウザからWebページを表示できない
- nginxを入れることで、HTTPでWebページを公開できる
- EC2を終了しても、画面上にしばらく「終了済み」として残ることがある

## 各ドキュメント

- [EC2インスタンス作成](docs/ec2_setup.md)
- [SSH接続](docs/ssh_connection.md)
- [セキュリティグループ設定](docs/security_group.md)
- [nginxセットアップ](docs/nginx_setup.md)

## 今後の改善案

- 独自のHTML/CSSページを配置する
- systemdやログ確認と組み合わせる
- セキュリティグループの設定変更による接続可否を検証する
- Nginxのアクセスログ・エラーログを確認する
- Elastic IPや独自ドメインの仕組みを学習する
