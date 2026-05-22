# AWS AMI Restore Lab

## 概要

このリポジトリは、EC2インスタンスからAMIを作成し、そのAMIを使って新しいEC2インスタンスを起動する実践記録です。

元のEC2にnginxをインストールし、Webページを作成した後、その状態をAMIとして保存しました。  
その後、作成したAMIから新しいEC2インスタンスを起動し、同じWebページが表示されることを確認しました。

---

## 目的

今回の目的は、AMIを使って「設定済みEC2を再利用できる」ことを理解することです。

```text
EC2を作成
↓
nginxをインストール
↓
Webページを作成
↓
AMIを作成
↓
AMIから新しいEC2を起動
↓
新しいEC2でも同じWebページを確認
```

---

## 今回のゴール

```text
設定済みのEC2をAMI化し、
そのAMIから新しいEC2を起動して、
同じWebページが表示されることを確認する
```

---

## 実践内容

今回実施した内容は以下です。

```text
元EC2インスタンスを作成
SSH接続
nginxインストール
index.html作成
ブラウザでWebページ表示確認
元EC2からAMI作成
AMIがavailableになることを確認
AMIから新しいEC2インスタンスを起動
新EC2でWebページ表示確認
AMIとUser Dataの違いを整理
リソース削除
```

---

## 構成イメージ

```text
元EC2
├── Ubuntu
├── nginx
└── /var/www/html/index.html

↓ AMI作成

自作AMI
└── 元EC2の中身を保存

↓ AMIから起動

新EC2
├── Ubuntu
├── nginx
└── /var/www/html/index.html
```

---

## 今回確認できたこと

```text
AMIはEC2の起動元イメージである
公式Ubuntu AMIの代わりに自作AMIを使える
AMIにはOS・インストール済みパッケージ・作成済みファイルなどが含まれる
AMIから起動した新EC2でも同じWebページを表示できる
AMI作成時には裏側でEBSスナップショットが作成される
AMI削除時はAMIの登録解除だけでなく、スナップショット削除も確認する
```

---

## AMIとは

AMIは、EC2インスタンスを起動するための元になるイメージです。

今までEC2作成時に選んでいたUbuntuも、正確には公式が用意しているUbuntu AMIです。

```text
公式Ubuntu AMI
→ まっさらなUbuntu EC2を起動する元ネタ

自作AMI
→ 自分で設定した状態のEC2を起動する元ネタ
```

今回作成したAMIは、nginxとWebページ作成済みのEC2を元にしたカスタムAMIです。

---

## AMIとスナップショットの関係

AMIを作成すると、裏側でルートEBSのスナップショットが作成されます。

```text
EC2
↓
AMI作成
↓
AMIが登録される
↓
裏側でEBSスナップショットが作成される
```

AMIはEC2起動用のテンプレートです。  
スナップショットはEBSボリュームのバックアップです。

---

## AMIとUser Dataの違い

今回、AMIとUser Dataの使い分けも整理しました。

```text
AMI
→ 完成済みの状態を保存して使う

User Data
→ 起動時にスクリプトで構築する
```

一言でまとめると以下です。

```text
AMIは「完成状態の保存」
User Dataは「起動時の自動作業」
```

---

## 使用した主なコマンド

元EC2で使用した主なコマンドです。

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl status nginx
echo "Hello from AMI practice instance" | sudo tee /var/www/html/index.html
curl localhost
```

AMIから起動した新EC2で確認した主なコマンドです。

```bash
systemctl status nginx
cat /var/www/html/index.html
curl localhost
```

---

## ドキュメント構成

```text
docs/ami-setup-steps.md
→ AMI作成までのセットアップ手順

docs/ami-after-launch-flow.md
→ AMIから新EC2を起動した後の確認手順

docs/ami-userdata-comparison.md
→ AMIとUser Dataの使い分け

docs/ami-learning-notes.md
→ AMI実践で学んだこと

docs/cleanup-checklist.md
→ 後片付けチェックリスト
```

---

## 後片付け

実践後、課金を抑えるために以下を削除しました。

```text
元EC2インスタンス
AMIから作成した新EC2インスタンス
作成したAMI
関連スナップショット
不要なEBSボリューム
```

Elastic IPは作成していないため、削除対応は不要でした。

---

## まとめ

今回の実践により、設定済みのEC2をAMIとして保存し、そのAMIから同じ状態のEC2を再作成できることを確認しました。

特に重要なポイントは以下です。

```text
AMIはEC2の中身を保存した起動元イメージ。
公式Ubuntu AMIの代わりに、自分で作ったAMIを使ってEC2を起動できる。
```

また、AMI作成時にはスナップショットが作成されるため、削除時はAMIだけでなくスナップショットも確認する必要があることを学びました。
