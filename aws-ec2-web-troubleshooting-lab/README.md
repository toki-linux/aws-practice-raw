# AWS EC2 Web Troubleshooting Lab

## 概要

AWS EC2上にWebサーバ環境を構築し、Webページが表示されない障害を意図的に再現・復旧した記録です。

このラボでは、以下の4つの障害パターンを扱いました。

- セキュリティグループによる外部アクセス不可
- nginxサービス停止
- Pythonバックエンド停止による502 Bad Gateway
- nginxの`proxy_pass`設定ミスによる404 Not Found

AWS側のネットワーク設定、EC2内部のサービス状態、nginxのリバースプロキシ設定、バックエンドアプリの状態を分けて確認し、ログとコマンドを使って原因を切り分けました。

---

## 目的

このラボの目的は、EC2上でWebサーバを構築するだけでなく、障害発生時にどこで問題が起きているのかを切り分ける力を身につけることです。

具体的には、以下を意識して実施しました。

- 外部からEC2へ通信が届いているか確認する
- nginxが起動しているか確認する
- 80番ポートで待ち受けているか確認する
- リバースプロキシ先のPythonサーバが起動しているか確認する
- nginxの設定ミスによるパスのずれを確認する
- `error.log`やコマンド結果をもとに原因を判断する

---

## 使用環境

| 項目 | 内容 |
|---|---|
| クラウド | AWS |
| サービス | EC2 |
| OS | Ubuntu |
| Webサーバ | nginx |
| バックエンド | Python http.server |
| 使用ポート | 80, 3000 |
| 主な確認コマンド | systemctl, ss, curl, tail, nginx -t |

---

## 構成

```text
外部ブラウザ
  ↓
AWS セキュリティグループ
  ↓
EC2 80番ポート
  ↓
nginx
  ↓
127.0.0.1:3000
  ↓
Python http.server
```

nginxでリバースプロキシを設定し、`/app/`へのアクセスをPythonサーバへ転送する構成にしました。

---

## 実施した障害パターン

| No | 障害内容 | 主な原因 | 主な確認ポイント | 詳細 |
|---|---|---|---|---|
| 01 | 外部からWebページにアクセスできない | セキュリティグループでHTTP 80番を許可していない | Security Group, curl, nginx状態 | [詳細](docs/01-security-group-http-closed.md) |
| 02 | Webページが表示されない | nginxサービス停止 | systemctl, ss, curl | [詳細](docs/02-nginx-stopped.md) |
| 03 | 502 Bad Gateway | Pythonバックエンド停止 | nginx error.log, ss, systemctl | [詳細](docs/03-python-backend-stopped-502.md) |
| 04 | 404 Not Found | proxy_passのパス設定ミス | nginx設定, curl, パスの渡り方 | [詳細](docs/04-proxy-pass-slash-404.md) |

---

## 障害ごとの切り分け比較

| 状態 | 外部アクセス | `curl localhost` | nginx | 80番 | Python | 3000番 | 主な原因 |
|---|---|---|---|---|---|---|---|
| 課題1 | NG | OK | 起動中 | LISTEN | - | - | セキュリティグループ |
| 課題2 | NG | NG | 停止中 | なし | - | - | nginx停止 |
| 課題3 | 502 | 502 | 起動中 | LISTEN | 停止中 | なし | Pythonサーバ停止 |
| 課題4 | 404 | 404 | 起動中 | LISTEN | 起動中 | LISTEN | proxy_pass設定ミス |

---

## よく使った確認コマンド

### nginxの状態確認

```bash
systemctl status nginx
```

### nginxの起動・停止・再起動

```bash
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
```

### ポートの待ち受け確認

```bash
ss -tulnp | grep :80
ss -tulnp | grep :3000
ss -tulnp | grep -E ':80|:3000'
```

### EC2内部からのHTTP確認

```bash
curl http://localhost
curl http://localhost/app/
curl http://127.0.0.1:3000
curl http://127.0.0.1:3000/app/
```

### nginx設定確認

```bash
sudo nginx -t
```

### nginx設定反映

```bash
sudo systemctl reload nginx
```

### nginxログ確認

```bash
sudo tail -n 30 /var/log/nginx/access.log
sudo tail -n 30 /var/log/nginx/error.log
```

---

## 全体を通して学んだこと

Webページが表示されない場合でも、原因は1つとは限りません。

今回のラボでは、同じ「Webページが表示されない」という症状でも、原因の場所によって確認すべきポイントが変わることを学びました。

```text
外部から通信が届かない
→ セキュリティグループを確認する

EC2内部でもアクセスできない
→ nginxの起動状態や80番ポートを確認する

nginxは動いているが502になる
→ バックエンドのPythonサーバを確認する

nginxもPythonも動いているが404になる
→ パスの渡し方や設定内容を確認する
```

障害対応では、エラー画面だけを見て判断するのではなく、通信経路を分解し、どこまで正常でどこから異常なのかを確認することが重要だと学びました。

---

## まとめ

このラボでは、AWS EC2上に構築したWebサーバ環境で、ネットワーク設定・nginx・バックエンドアプリ・リバースプロキシ設定に関する障害を再現しました。

それぞれの障害について、症状、原因、確認コマンド、復旧手順を整理し、Webサーバ障害の基本的な切り分け手順を学びました。

単にEC2を起動してWebページを表示するだけでなく、障害発生時にログとコマンドを使って原因を確認し、復旧まで行う経験を積むことができました。

## 追加学習：EC2のAWS独自機能

トラブルシューティング課題の後、EC2特有の機能として以下を学習しました。

| No | 内容 | 学んだこと | 詳細 |
|---|---|---|---|
| 05 | EC2 User Data | インスタンス初回起動時に初期設定を自動実行する仕組み | [詳細](docs/05-ec2-user-data.md) |
| 06 | EC2 Instance Metadata | EC2自身の情報をインスタンス内から取得する仕組み | [詳細](docs/06-ec2-instance-metadata.md) |

---

## EC2独自機能で学んだこと

これまでのLinuxやnginxの操作は、VirtualBoxなどのローカル仮想環境でも再現できます。

一方で、今回扱ったUser DataやInstance Metadataは、AWS EC2ならではの機能です。

```text
User Data
→ EC2作成時に初期設定を自動化する

Instance Metadata
→ EC2自身の情報をインスタンス内から取得する
```

これにより、単にLinuxサーバを操作するだけでなく、AWS上のEC2としてどのように初期構築・情報取得を行うのかを学びました。
