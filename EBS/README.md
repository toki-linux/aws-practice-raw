# EBS Additional Volume Mount Lab

## 概要

EC2インスタンスに追加EBSボリュームを作成・アタッチし、Ubuntu上で `/data` として利用できるようにした実践。

追加EBSを手動マウントした後、`/etc/fstab` を設定し、再起動後も自動マウントされることを確認した。

---

## 実践内容

今回実施した内容は以下。

```text
EC2インスタンス作成
追加EBSボリューム作成
EC2へEBSをアタッチ
Ubuntu上でlsblk確認
ファイルシステム作成
/dataディレクトリ作成
/dataへマウント
/data/test.txt作成
UUID確認
/etc/fstab設定
mount -aで設定確認
再起動後の自動マウント確認
リソース削除
```

---

## 構成イメージ

```text
EC2
├── ルートEBS
│   └── /
│       ├── /home
│       ├── /etc
│       └── /var
│
└── 追加EBS
    └── /data
        └── test.txt
```

---

## 今回のゴール

```text
追加EBSを /data にマウントし、
再起動後も /data/test.txt を読める状態にする
```

---

## 使用した主なコマンド

```bash
lsblk
df -h
sudo file -s /dev/nvme1n1
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
echo "hello from additional ebs" | sudo tee /data/test.txt
cat /data/test.txt
sudo blkid /dev/nvme1n1
sudo cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo reboot
```

---

## 重要ポイント

```text
アタッチされている
= EC2にEBSが接続されている

マウントされている
= Linux上のディレクトリとして使える

自動マウントされている
= 再起動後も自動で使える
```

特に重要な切り分け。

```text
lsblkに出る
→ ディスクとして認識されている

df -hに出る
→ マウントされ、保存場所として使える
```

つまり、

```text
lsblkに nvme1n1 がある
df -hに /data がない

→ 追加EBSは認識されているが、まだマウントされていない
```

---

## ドキュメント

```text
docs/ebs-create-steps.md
→ EBS追加ボリューム作成手順

docs/ebs-after-attach-flow.md
→ EBSアタッチ後のUbuntu側作業フロー

docs/ebs-commands-used.md
→ 使用コマンドまとめ

docs/ebs-learning-notes.md
→ 学びのまとめ

docs/ebs-troubleshooting.md
→ トラブルシューティング

docs/ebs-cleanup-checklist.md
→ 後片付けチェックリスト
```

---

## 今回の学び

```text
EBSはEC2に接続して使うブロックストレージ
EC2にはルートEBSが最初から付いている
追加EBSはデータ保存用ディスクとして使える
アタッチしただけではLinux上の保存場所としては使えない
ファイルシステムを作ることでUbuntuがファイルを管理できる形式になる
マウントすることで /data として使えるようになる
/etc/fstab に設定することで再起動後も自動マウントできる
```

---

## 後片付け

実践後、課金を抑えるために以下を削除した。

```text
EC2インスタンス
追加EBSボリューム
```

スナップショットとElastic IPは作成していないため、削除対応は不要。

セキュリティグループは課金対象ではないため、必要に応じて残しておく。
