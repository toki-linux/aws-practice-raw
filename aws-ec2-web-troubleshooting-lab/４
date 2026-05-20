# 課題4：proxy_passの末尾スラッシュを外して404を再現する

## 概要

AWS EC2上で動作しているnginxのリバースプロキシ設定を変更し、  
`proxy_pass` の末尾スラッシュの有無によって、バックエンドへ渡されるパスが変わることを確認した。

nginxとPythonサーバはどちらも起動している状態で、  
パスの渡し方がずれることにより404 Not Foundが発生する状態を再現した。

## 症状

外部ブラウザから `/app/` にアクセスすると、  
404 Not Found が表示された。

## 原因

nginxの `proxy_pass` の末尾スラッシュを外したことで、  
リクエストされた `/app/` がそのままPythonサーバ側へ渡された。

Pythonサーバ側には `/app/` に対応するファイルやディレクトリが存在しなかったため、  
404 Not Found となった。

## 変更した設定

正常時の設定：

```nginx
location /app/ {
    proxy_pass http://127.0.0.1:3000/;
}

障害再現時の設定：

location /app/ {
    proxy_pass http://127.0.0.1:3000;
}
確認コマンド

nginxの設定に構文エラーがないか確認した。

sudo nginx -t

nginx設定を反映した。

sudo systemctl reload nginx

nginxとPythonサーバの待ち受け状態を確認した。

ss -tulnp | grep -E ':80|:3000'

nginx経由で /app/ にアクセスした。

curl http://localhost/app/

Pythonサーバへ直接アクセスした。

curl http://127.0.0.1:3000

Pythonサーバ側に /app/ が存在するか確認した。

curl http://127.0.0.1:3000/app/
確認できたこと

nginxは起動しており、80番ポートで待ち受けていた。

Pythonサーバも起動しており、3000番ポートで待ち受けていた。

しかし、proxy_pass の末尾スラッシュを外したことで、
/app/ というパスがPythonサーバ側へそのまま渡され、
Pythonサーバ側で404 Not Foundとなった。

解決

proxy_pass の末尾にスラッシュを戻した。

location /app/ {
    proxy_pass http://127.0.0.1:3000/;
}

その後、nginxの設定確認と反映を行った。

sudo nginx -t
sudo systemctl reload nginx

再度 /app/ にアクセスし、Webページが表示されることを確認した。

学び

404 Not Found は、ファイルが単純に存在しない場合だけでなく、
リバースプロキシでバックエンドへ渡すパスがずれた場合にも発生する。

今回のように、nginxとPythonサーバがどちらも起動していても、
proxy_pass の書き方によってバックエンドへ渡されるパスが変わる。

そのため、404が発生した場合は、
ファイルの存在確認だけでなく、nginxの location と proxy_pass の対応関係も確認する必要がある。
