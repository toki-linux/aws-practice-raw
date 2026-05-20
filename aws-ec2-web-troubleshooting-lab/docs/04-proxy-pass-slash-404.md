# 課題4：proxy_passの末尾スラッシュを外して404を再現する

## 概要

AWS EC2上で動作しているnginxのリバースプロキシ設定を変更し、`proxy_pass`の末尾スラッシュの有無によって、バックエンドへ渡されるパスが変わることを確認しました。

nginxとPythonサーバはどちらも起動している状態で、パスの渡し方がずれることにより404 Not Foundが発生する状態を再現しました。

---

## 症状

外部ブラウザから`/app/`にアクセスすると、404 Not Foundが表示されました。

---

## 原因

nginxの`proxy_pass`の末尾スラッシュを外したことで、リクエストされた`/app/`がそのままPythonサーバ側へ渡されました。

Pythonサーバ側には`/app/`に対応するファイルやディレクトリが存在しなかったため、404 Not Foundとなりました。

---

## 変更した設定

正常時の設定：

```nginx
location /app/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

障害再現時の設定：

```nginx
location /app/ {
    proxy_pass http://127.0.0.1:3000;
}
```

違いは、`proxy_pass`の末尾に`/`があるかどうかです。

```text
正常時：proxy_pass http://127.0.0.1:3000/;
障害時：proxy_pass http://127.0.0.1:3000;
```

---

## パスの渡り方

正常時の設定では、`/app/`が取り除かれてPythonサーバ側へ渡されます。

```text
ブラウザからのアクセス：/app/
Pythonサーバ側への渡り方：/
```

一方、末尾スラッシュを外した設定では、`/app/`がそのままPythonサーバ側へ渡されます。

```text
ブラウザからのアクセス：/app/
Pythonサーバ側への渡り方：/app/
```

Pythonサーバ側に`/app/`というディレクトリやファイルが存在しないため、404 Not Foundとなりました。

---

## 切り分け

nginxの設定に構文エラーがないか確認しました。

```bash
sudo nginx -t
```

nginx設定を反映しました。

```bash
sudo systemctl reload nginx
```

nginxとPythonサーバの待ち受け状態を確認しました。

```bash
ss -tulnp | grep -E ':80|:3000'
```

nginx経由で`/app/`にアクセスしました。

```bash
curl http://localhost/app/
```

Pythonサーバへ直接アクセスしました。

```bash
curl http://127.0.0.1:3000
```

Pythonサーバ側に`/app/`が存在するか確認しました。

```bash
curl http://127.0.0.1:3000/app/
```

---

## 確認できたこと

nginxは起動しており、80番ポートで待ち受けていました。

Pythonサーバも起動しており、3000番ポートで待ち受けていました。

しかし、`proxy_pass`の末尾スラッシュを外したことで、`/app/`というパスがPythonサーバ側へそのまま渡され、Pythonサーバ側で404 Not Foundとなりました。

```text
nginx停止        → していない
Python停止       → していない
3000番ポート停止 → していない
パスの渡し方      → ずれていた
```

---

## 課題4のポイント

課題4では、nginxもPythonサーバも停止していません。

それでも404が発生しました。

理由は、リバースプロキシの設定によってPythonサーバ側へ渡されるパスが変わったためです。

```text
nginxは動いている
Pythonサーバも動いている
80番も3000番もLISTENしている
でも、渡されるパスがずれている
そのため404になる
```

課題3との違いは以下です。

```text
課題3：Pythonサーバが停止しているため502
課題4：Pythonサーバは動いているが、渡されるパスがずれて404
```

---

## 解決

`proxy_pass`の末尾にスラッシュを戻しました。

```nginx
location /app/ {
    proxy_pass http://127.0.0.1:3000/;
}
```

その後、nginxの設定確認と反映を行いました。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

再度`/app/`にアクセスし、Webページが表示されることを確認しました。

```bash
curl http://localhost/app/
```

外部ブラウザからも`/app/`にアクセスし、復旧を確認しました。

---

## 学び

404 Not Foundは、ファイルが単純に存在しない場合だけでなく、リバースプロキシでバックエンドへ渡すパスがずれた場合にも発生します。

今回のように、nginxとPythonサーバがどちらも起動していても、`proxy_pass`の書き方によってバックエンドへ渡されるパスが変わります。

そのため、404が発生した場合は、ファイルの存在確認だけでなく、nginxの`location`と`proxy_pass`の対応関係も確認する必要があると学びました。
