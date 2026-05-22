# AMIから新EC2を起動した後の確認手順

## 目的

このドキュメントでは、作成したAMIから新しいEC2インスタンスを起動し、元EC2と同じ状態が再現されているか確認する手順をまとめます。

今回確認したいことは以下です。

```text
AMIから起動した新EC2に、
nginxとindex.htmlが最初から入っていること
```

---

## 全体の流れ

```text
作成したAMIを選択
↓
AMIから新しいEC2を起動
↓
新EC2のパブリックIPを確認
↓
ブラウザでWebページ確認
↓
SSH接続
↓
nginxの状態確認
↓
index.htmlの中身確認
↓
curl localhostで確認
```

---

## 1. 作成したAMIを選択する

作成したAMIを選択しました。

確認場所です。

```text
EC2
→ イメージ
→ AMI
```

作成したAMIが `available` になっていることを確認します。

---

## 2. AMIからインスタンスを起動する

作成したAMIを選択し、そこから新しいEC2インスタンスを起動しました。

操作です。

```text
AMIを選択
→ AMIからインスタンスを起動
```

---

## 3. 新EC2の設定を行う

AMIから起動する場合でも、以下の項目は改めて設定します。

```text
インスタンスタイプ
キーペア
セキュリティグループ
サブネット
パブリックIP
IAM Role
```

今回の設定例です。

```text
名前：
ami-restore-test

インスタンスタイプ：
低コスト系

キーペア：
SSH接続用の既存キー

セキュリティグループ：
SSH 22番：自分のIP
HTTP 80番：許可
```

---

## 4. User Dataは設定しない

今回はUser Dataを設定しませんでした。

理由は、AMIの中にすでにnginxとindex.htmlが入っているか確認したいからです。

```text
User Dataで構築する
→ 起動時にスクリプトでセットアップする

AMIから起動する
→ 構築済み状態から起動する
```

今回確認したいのは後者です。

---

## 5. 新EC2のパブリックIPを確認する

新しく起動したEC2インスタンスのパブリックIPを確認しました。

確認場所です。

```text
EC2
→ インスタンス
→ 新しく起動したインスタンス
→ パブリックIPv4アドレス
```

---

## 6. ブラウザでWebページを確認する

ブラウザで以下にアクセスしました。

```text
http://新EC2のパブリックIP
```

元EC2で作成したWebページが表示されれば成功です。

期待する表示です。

```text
Hello from AMI practice instance
```

---

## 7. SSH接続する

必要に応じて、新EC2へSSH接続します。

```bash
ssh -i キー名.pem ubuntu@新EC2のパブリックIP
```

---

## 8. nginxの状態を確認する

新EC2上で、nginxが存在し、起動しているか確認します。

```bash
systemctl status nginx
```

`active (running)` になっていれば、nginxが起動しています。

---

## 9. index.htmlの中身を確認する

元EC2で作成した `index.html` が存在するか確認します。

```bash
cat /var/www/html/index.html
```

期待する表示です。

```text
Hello from AMI practice instance
```

---

## 10. curl localhostで確認する

EC2内部からWebページを確認します。

```bash
curl localhost
```

期待する表示です。

```text
Hello from AMI practice instance
```

---

## 11. 成功状態

今回の成功状態は以下です。

```text
AMIから起動した新EC2で、
元EC2と同じWebページが表示された
```

つまり、以下の流れが成功しました。

```text
元EC2でWebサーバを構築
↓
AMIを作成
↓
AMIから新EC2を起動
↓
新EC2でも同じ状態を再現
```

---

## 今回理解したこと

AMIから起動する新EC2は、公式Ubuntu AMIではなく、自分で作成したカスタムAMIを元に起動しています。

```text
今まで：
公式Ubuntu AMIからEC2を起動

今回：
自作AMIからEC2を起動
```

そのため、元EC2に入っていたnginxやindex.htmlが、新EC2にも入った状態で起動しました。

---

## AMIから起動しても改めて選ぶもの

AMIはEC2の中身の元ネタですが、以下はAMIから起動するときに改めて選びます。

```text
インスタンスタイプ
キーペア
セキュリティグループ
サブネット
パブリックIP
IAM Role
```

これらは、AMIに固定されるものではなく、EC2起動時に指定するAWS側の設定です。

---

## まとめ

今回、AMIから新しいEC2を起動し、元EC2と同じWebページを表示できました。

これにより、AMIを使うことで、設定済みEC2を再利用・複製できることを確認しました。
