# 課題3：Pythonサーバを停止して502 Bad Gatewayを再現する

## 概要

AWS EC2上で、nginxのリバースプロキシ先として動作しているPythonサーバを停止し、  
502 Bad Gateway が発生する状態を再現した。

nginx自体は起動している状態で、バックエンドのPythonサーバだけを停止させることで、  
Webサーバとバックエンドアプリの切り分けを行った。

## 症状

外部ブラウザから `/app/` にアクセスすると、  
502 Bad Gateway が表示された。

## 原因

nginxのリバースプロキシ先であるPythonサーバが停止していた。

そのため、nginxはリクエストを受け取ることはできたが、  
バックエンドの `127.0.0.1:3000` に接続できず、502エラーとなった。

## 確認コマンド

nginxの状態を確認した。

```bash
systemctl status nginx

80番ポートと3000番ポートの待ち受け状態を確認した。

ss -tulnp | grep -E ':80|:3000'

nginx経由で /app/ にアクセスした。

curl http://localhost/app/

Pythonサーバに直接アクセスできるか確認した。

curl http://127.0.0.1:3000

nginxのerror.logを確認した。

sudo tail -n 30 /var/log/nginx/error.log
確認できたこと

nginxは起動しており、80番ポートで待ち受けていた。

一方で、Pythonサーバを停止したため、
3000番ポートでは待ち受けがなくなっていた。

その結果、nginxはバックエンドに接続できず、
502 Bad Gateway が発生した。

解決

停止していたPythonサーバを起動し直した。

systemdで管理している場合：

sudo systemctl start myapp

手動起動の場合：

python3 -m http.server 3000 --bind 127.0.0.1

その後、再度 /app/ にアクセスし、
Webページが表示されることを確認した。

学び

502 Bad Gateway は、nginx自体の停止ではなく、
nginxの先にあるバックエンドサーバへ接続できない場合にも発生する。

今回のように、80番ポートではnginxが待ち受けていても、
3000番ポートのPythonサーバが停止していれば、
nginxはリクエストを処理しきれず502を返す。

そのため、502が発生した場合は、
nginxの状態だけでなく、リバースプロキシ先のアプリケーションやポートの状態も確認する必要がある。
