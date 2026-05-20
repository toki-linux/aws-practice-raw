# 課題3：Pythonサーバを停止して502 Bad Gatewayを再現する

## 概要

AWS EC2上で、nginxのリバースプロキシ先として動作しているPythonサーバを停止し、502 Bad Gatewayが発生する状態を再現しました。

nginx自体は起動している状態で、バックエンドのPythonサーバだけを停止させることで、Webサーバとバックエンドアプリの切り分けを行いました。

---

## 症状

外部ブラウザから`/app/`にアクセスすると、502 Bad Gatewayが表示されました。

---

## 原因

nginxのリバースプロキシ先であるPythonサーバが停止していました。

そのため、nginxはリクエストを受け取ることはできましたが、バックエンドの`127.0.0.1:3000`に接続できず、502エラーとなりました。

---

## 構成

```text
外部ブラウザ
  ↓
EC2 80番ポート
  ↓
nginx
  ↓
127.0.0.1:3000
  ↓
Python http.server
```

この課題では、nginxは起動したまま、リバースプロキシ先のPythonサーバだけを停止しました。

---

## 実施したこと

Pythonサーバを停止しました。

systemdで管理している場合：

```bash
sudo systemctl stop myapp
```

手動で起動している場合は、プロセスを確認して停止します。

```bash
ps aux | grep http.server
kill PID番号
```

---

## 切り分け

nginxの状態を確認しました。

```bash
systemctl status nginx
```

80番ポートと3000番ポートの待ち受け状態を確認しました。

```bash
ss -tulnp | grep -E ':80|:3000'
```

nginx経由で`/app/`にアクセスしました。

```bash
curl http://localhost/app/
```

Pythonサーバに直接アクセスできるか確認しました。

```bash
curl http://127.0.0.1:3000
```

nginxの`error.log`を確認しました。

```bash
sudo tail -n 30 /var/log/nginx/error.log
```

`error.log`では、nginxがバックエンドへ接続できていないことを確認しました。

```text
connect() failed (111: Connection refused) while connecting to upstream
```

---

## 確認できたこと

nginxは起動しており、80番ポートで待ち受けていました。

一方で、Pythonサーバを停止したため、3000番ポートでは待ち受けがなくなっていました。

```text
80番   → nginxがLISTENしている
3000番 → PythonサーバがLISTENしていない
```

その結果、nginxはバックエンドに接続できず、502 Bad Gatewayが発生しました。

---

## 課題3のポイント

課題3では、nginx自体は正常に動作しています。

しかし、nginxの先にあるPythonサーバが停止しているため、nginxはリクエストを受け取ったあとにバックエンドへ接続できません。

```text
ブラウザ
  ↓
nginxまでは届く
  ↓
Pythonサーバへ接続しようとする
  ↓
Pythonサーバが停止している
  ↓
502 Bad Gateway
```

課題2との違いは以下です。

```text
課題2：nginx自体が停止している
課題3：nginxは起動しているが、バックエンドが停止している
```

---

## 解決

停止していたPythonサーバを起動し直しました。

systemdで管理している場合：

```bash
sudo systemctl start myapp
```

手動起動の場合：

```bash
python3 -m http.server 3000 --bind 127.0.0.1
```

その後、ポートの待ち受けを確認しました。

```bash
ss -tulnp | grep -E ':80|:3000'
```

nginx経由で`/app/`にアクセスし、Webページが表示されることを確認しました。

```bash
curl http://localhost/app/
```

外部ブラウザからも`/app/`にアクセスし、復旧を確認しました。

---

## 学び

502 Bad Gatewayは、nginx自体の停止ではなく、nginxの先にあるバックエンドサーバへ接続できない場合にも発生します。

今回のように、80番ポートではnginxが待ち受けていても、3000番ポートのPythonサーバが停止していれば、nginxはリクエストを処理しきれず502を返します。

そのため、502が発生した場合は、nginxの状態だけでなく、リバースプロキシ先のアプリケーションやポートの状態も確認する必要があると学びました。
