# nginxセットアップ

## 概要

EC2上のUbuntuサーバにnginxをインストールし、Webページを表示できるようにした。

## nginxのインストール

以下のコマンドでパッケージ情報を更新した。

```bash
sudo apt update
```

nginxをインストールした。

```bash
sudo apt install nginx -y
```

## nginxの起動確認

```bash
systemctl status nginx
```

`active (running)` と表示されれば、nginxは起動している。

## Webページ配置場所

nginxのデフォルトの公開ディレクトリとして、以下を使用した。

```text
/var/www/html
```

## index.htmlの作成

以下のような内容で `index.html` を作成した。

```html
<h1>Hello from EC2</h1>
<p>My first AWS web server.</p>
```

例：

```bash
echo "<h1>Hello from EC2</h1><p>My first AWS web server.</p>" | sudo tee /var/www/html/index.html
```

## サーバ内での確認

EC2内から以下のコマンドで確認した。

```bash
curl http://localhost
```

作成したHTMLが返ってくれば、nginxがローカルで正常に応答している。

## ブラウザでの確認

自分のPCのブラウザから以下にアクセスした。

```text
http://パブリックIPv4アドレス
```

作成したWebページが表示されれば成功。

## トラブル時の確認ポイント

Webページが表示されない場合は、以下を確認する。

```bash
systemctl status nginx
curl http://localhost
ss -tulnp | grep ':80'
```

確認する観点は以下。

- nginxが起動しているか
- 80番ポートで待ち受けているか
- `/var/www/html/index.html` が存在するか
- セキュリティグループでHTTP 80番が許可されているか
- ブラウザで `http://` を使っているか

## 学んだこと

- nginxをインストールするとWebサーバとしてHTTP通信を受け付けられる
- `/var/www/html/index.html` にファイルを置くと、ブラウザから表示できる
- サーバ内では `curl http://localhost` で確認できる
- 外部ブラウザから見るには、AWS側のセキュリティグループで80番ポートを許可する必要がある
