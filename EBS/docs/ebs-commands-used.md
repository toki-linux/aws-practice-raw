# EBS追加ボリューム実践：使用コマンドまとめ

## 目的

このドキュメントでは、EC2に追加EBSをアタッチし、Ubuntu上で `/data` として使えるようにするまでに使用したコマンドを整理する。

---

## 1. SSH接続

```bash
ssh -i キー名.pem ubuntu@パブリックIP
```

### 目的

EC2インスタンスへSSH接続する。

### 意味

```text
ssh
→ リモートサーバへ接続する

-i キー名.pem
→ SSH接続に使う秘密鍵を指定する

ubuntu
→ Ubuntu AMIのログインユーザー

パブリックIP
→ EC2インスタンスのパブリックIPアドレス
```

---

## 2. ディスク一覧を確認する

```bash
lsblk
```

### 目的

Ubuntuが認識しているディスクやパーティションを確認する。

### 今回のポイント

```text
nvme0n1
→ ルートEBS

nvme1n1
→ 追加EBS
```

### 見方

```text
MOUNTPOINTSに / がある
→ OSで使っているルートディスク

MOUNTPOINTSが空
→ まだマウントされていないディスク
```

### 重要な理解

```text
lsblkに表示される
→ Linuxがディスクの存在を認識している

ただし、まだ使える保存場所とは限らない
```

---

## 3. マウント済みのファイルシステムを確認する

```bash
df -h
```

### 目的

実際にマウントされ、保存場所として使えるファイルシステムを確認する。

### 意味

```text
df
→ ディスク使用量を表示する

-h
→ 人間が読みやすい単位で表示する
```

### lsblkとの違い

```text
lsblk
→ ディスクとして見えているか確認する

df -h
→ マウントされて使える状態か確認する
```

### 今回の重要ポイント

```text
lsblkに nvme1n1 がある
df -hに /data がない

→ 追加EBSは認識されているが、まだマウントされていない
```

---

## 4. ファイルシステムの有無を確認する

```bash
sudo file -s /dev/nvme1n1
```

### 目的

追加EBSにファイルシステムがあるか確認する。

### 意味

```text
sudo
→ 管理者権限で実行する

file
→ ファイルやデバイスの種類を確認する

-s
→ デバイスファイルの中身も確認する

/dev/nvme1n1
→ 追加EBSとして認識されたディスク
```

### 結果例

ファイルシステムがない場合。

```text
/dev/nvme1n1: data
```

ファイルシステムがある場合。

```text
/dev/nvme1n1: Linux rev 1.0 ext4 filesystem data ...
```

---

## 5. ファイルシステムを作成する

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

### 目的

追加EBSをUbuntuがファイルを保存できる形式にする。

### 意味

```text
mkfs
→ make filesystem
→ ファイルシステムを作成する

-t ext4
→ ext4形式で作成する

/dev/nvme1n1
→ 対象の追加EBS
```

### 重要な注意点

```text
mkfsは初期化コマンド
対象を間違えるとデータが消える
```

ルートEBSではなく、追加EBSである `/dev/nvme1n1` を指定する。

### 今回の理解

```text
ファイルシステムを作る
= Ubuntuがそのディスクにファイルやディレクトリを保存できる形式にする
```

---

## 6. マウント先ディレクトリを作成する

```bash
sudo mkdir /data
```

### 目的

追加EBSを接続するためのディレクトリを作成する。

### 意味

```text
mkdir
→ ディレクトリを作成する

/data
→ 追加EBSをマウントする場所
```

---

## 7. 追加EBSを /data にマウントする

```bash
sudo mount /dev/nvme1n1 /data
```

### 目的

追加EBSを `/data` ディレクトリとして使えるようにする。

### 意味

```text
mount
→ ディスクやファイルシステムをディレクトリに接続する

/dev/nvme1n1
→ 接続元の追加EBS

/data
→ 接続先のディレクトリ
```

### 今回の理解

```text
アタッチ
→ EC2にEBSを接続する

マウント
→ Linux上のディレクトリとして使えるようにする
```

---

## 8. マウント確認

```bash
df -h
```

または、

```bash
lsblk
```

### 目的

追加EBSが `/data` にマウントされているか確認する。

### 期待する状態

```text
/dev/nvme1n1 が /data にマウントされている
```

---

## 9. /dataにファイルを作成する

```bash
echo "hello from additional ebs" | sudo tee /data/test.txt
```

### 目的

追加EBS上にファイルを書き込めるか確認する。

### 意味

```text
echo
→ 文字列を表示する

|
→ 左の出力を右のコマンドへ渡す

sudo tee /data/test.txt
→ 管理者権限で /data/test.txt に書き込む
```

---

## 10. ファイルの中身を確認する

```bash
cat /data/test.txt
```

### 目的

追加EBS上に作成したファイルを読めるか確認する。

### 意味

```text
cat
→ ファイルの中身を表示する

/data/test.txt
→ 追加EBS上に作成したファイル
```

---

## 11. UUIDを確認する

```bash
sudo blkid /dev/nvme1n1
```

### 目的

自動マウント設定に使うUUIDを確認する。

### 意味

```text
blkid
→ ブロックデバイスのUUIDやファイルシステム種別を表示する

/dev/nvme1n1
→ 確認対象の追加EBS
```

### 結果例

```text
/dev/nvme1n1: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" BLOCK_SIZE="4096" TYPE="ext4"
```

---

## 12. /etc/fstabをバックアップする

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

### 目的

自動マウント設定ファイルを編集する前にバックアップを取る。

### 意味

```text
cp
→ ファイルをコピーする

/etc/fstab
→ 起動時のマウント設定ファイル

/etc/fstab.bak
→ バックアップ先
```

---

## 13. /etc/fstabを編集する

```bash
sudo nano /etc/fstab
```

### 目的

再起動後も `/data` に自動マウントされるように設定する。

### 追記した内容

```text
UUID=自分のUUID /data ext4 defaults,nofail 0 2
```

### 各項目の意味

```text
UUID=自分のUUID
→ 対象のEBSをUUIDで指定する

/data
→ マウント先

ext4
→ ファイルシステムの種類

defaults,nofail
→ 標準設定 + 見つからなくても起動を止めない

0
→ dumpバックアップ対象外

2
→ 起動時のファイルシステムチェック順序
```

---

## 14. systemdに設定変更を読み直させる

```bash
sudo systemctl daemon-reload
```

### 目的

`/etc/fstab` の変更をsystemdに読み直させる。

### 今回出たメッセージ

```text
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

この表示が出たため、`systemctl daemon-reload` を実行した。

---

## 15. fstabの設定をテストする

```bash
sudo mount -a
```

### 目的

`/etc/fstab` に書いた内容でマウントできるか確認する。

### 意味

```text
mount -a
→ /etc/fstab に書かれているものをまとめてマウントする
```

### ポイント

```text
何も表示されない
→ 基本的に成功

エラーが出る
→ /etc/fstab の書き方やUUIDを確認する
```

いきなり再起動する前に、`mount -a` で確認する。

---

## 16. 再起動する

```bash
sudo reboot
```

### 目的

再起動後も `/data` が自動マウントされるか確認する。

---

## 17. 再起動後に確認する

再起動後、再度SSH接続する。

```bash
ssh -i キー名.pem ubuntu@パブリックIP
```

確認コマンド。

```bash
lsblk
df -h
cat /data/test.txt
```

### 確認すること

```text
lsblk
→ nvme1n1 が存在しているか

df -h
→ /data がマウントされているか

cat /data/test.txt
→ 追加EBS上のファイルが読めるか
```

---

## 18. 今回使用したコマンド一覧

```bash
ssh -i キー名.pem ubuntu@パブリックIP
```

```bash
lsblk
```

```bash
df -h
```

```bash
sudo file -s /dev/nvme1n1
```

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

```bash
sudo mkdir /data
```

```bash
sudo mount /dev/nvme1n1 /data
```

```bash
df -h
```

```bash
lsblk
```

```bash
echo "hello from additional ebs" | sudo tee /data/test.txt
```

```bash
cat /data/test.txt
```

```bash
sudo blkid /dev/nvme1n1
```

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

```bash
sudo nano /etc/fstab
```

```bash
sudo systemctl daemon-reload
```

```bash
sudo mount -a
```

```bash
sudo reboot
```

```bash
lsblk
df -h
cat /data/test.txt
```

---

## 19. 今回のコマンド全体の流れ

```text
lsblk
→ 追加EBSが見えているか確認

file -s
→ ファイルシステムがあるか確認

mkfs
→ ファイルシステムを作る

mkdir
→ マウント先を作る

mount
→ /dataにマウントする

df -h
→ マウントされたか確認

tee /data/test.txt
→ ファイル作成テスト

cat /data/test.txt
→ 中身確認

blkid
→ UUID確認

fstab編集
→ 自動マウント設定

daemon-reload
→ systemdに設定変更を反映

mount -a
→ fstab設定をテスト

reboot
→ 再起動確認

df -h / cat
→ 再起動後もマウント・ファイル確認
```

---

## 20. 重要な切り分け

```text
lsblkに出る
→ ディスクとして認識されている

df -hに出る
→ マウントされ、保存場所として使える

cat /data/test.txt が読める
→ 追加EBS上のデータが使えている
```

今回一番重要な状態。

```text
lsblkに nvme1n1 がある
df -hに /data がない

→ アタッチはされているが、マウントされていない
```
