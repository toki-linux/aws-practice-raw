# EBS追加ボリューム実践：トラブルシューティング

## 目的

このドキュメントでは、EBS追加ボリューム実践中に確認した状態や、つまずきやすいポイントを整理する。

---

## 1. lsblkには出るがdf -hには出ない

### 状況

`lsblk` では追加EBSと思われる `nvme1n1` が見える。

```text
nvme1n1  259:5  0  1G  0 disk
```

しかし、`df -h` では `/data` が表示されない。

### 意味

```text
lsblkに出る
→ Linuxはディスクの存在を認識している

df -hに出ない
→ まだマウントされていない
```

つまり、

```text
追加EBSはアタッチされているが、
Linux上の保存場所としてはまだ使えない
```

という状態。

### 対応

まだマウントしていない場合は、以下を実行する。

```bash
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
```

確認。

```bash
df -h
lsblk
```

---

## 2. 追加EBSがどれか分からない

### 状況

`lsblk` を実行すると複数のディスクが表示される。

例。

```text
nvme0n1      259:0    0    8G  0 disk
├─nvme0n1p1  259:1    0    7G  0 part /
├─nvme0n1p15 259:3    0  106M  0 part /boot/efi
└─nvme0n1p16 259:4    0  913M  0 part /boot
nvme1n1      259:5    0    1G  0 disk
```

### 判断方法

```text
nvme0n1
→ ルートEBS
→ / や /boot が付いている

nvme1n1
→ 追加EBS
→ 今回作成した1GiB
→ マウントポイントがない
```

### 今回の判断

```text
サイズが1G
マウントポイントがない
パーティションがない
今回作ったEBSのサイズと一致
```

このため、`nvme1n1` を追加EBSと判断した。

---

## 3. file -sで data と表示される

### コマンド

```bash
sudo file -s /dev/nvme1n1
```

### 結果

```text
/dev/nvme1n1: data
```

### 意味

まだファイルシステムが作成されていない状態。

Ubuntuがファイルやディレクトリを保存・管理できる形式になっていない。

### 対応

ext4でファイルシステムを作成する。

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

---

## 4. mkfsを再実行してはいけない場面

### 状況

一度シャットダウンや再起動をした後、`df -h` に `/data` が表示されない。

しかし、`lsblk` には `nvme1n1` が表示されている。

### 注意

このとき、すぐに `mkfs` を実行してはいけない。

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

`mkfs` はファイルシステムを作成するコマンドだが、既存データを初期化する操作になる。

すでに前回ファイルを作っている場合、再度 `mkfs` するとデータが消える。

### 先に確認すること

```bash
sudo file -s /dev/nvme1n1
```

ext4と表示される場合。

```text
/dev/nvme1n1: Linux rev 1.0 ext4 filesystem data ...
```

この場合は、すでにファイルシステム作成済み。

### 対応

フォーマットせず、マウントする。

```bash
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
```

確認。

```bash
cat /data/test.txt
```

---

## 5. /etc/fstab変更後にsystemdの警告が出る

### 状況

`/etc/fstab` を変更後、`mount -a` を実行したときに以下のような表示が出た。

```text
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

### 意味

`/etc/fstab` は変更されたが、systemdがまだ古い設定を使っている。

### 対応

systemdに設定を読み直させる。

```bash
sudo systemctl daemon-reload
```

その後、再度確認する。

```bash
sudo mount -a
```

---

## 6. mount -aでエラーが出た場合

### 状況

`/etc/fstab` に追記した後、以下を実行した。

```bash
sudo mount -a
```

ここでエラーが出る場合、fstabの書き方に問題がある可能性がある。

### 確認すること

```text
UUIDが正しいか
/data が存在するか
ファイルシステム種別が ext4 になっているか
スペース区切りが正しいか
余計な文字が入っていないか
```

### 確認コマンド

UUID確認。

```bash
sudo blkid /dev/nvme1n1
```

マウント先確認。

```bash
ls -ld /data
```

fstab確認。

```bash
cat /etc/fstab
```

---

## 7. 再起動後に/dataが見えない

### 状況

再起動後、`df -h` に `/data` が出ない。

### 考えられる原因

```text
/etc/fstabに追記できていない
UUIDが間違っている
/dataディレクトリがない
mount -aで事前確認していない
追加EBSがアタッチされていない
```

### 確認順

```bash
lsblk
```

```bash
df -h
```

```bash
sudo blkid /dev/nvme1n1
```

```bash
cat /etc/fstab
```

```bash
sudo mount -a
```

---

## 8. /data/test.txt が読めない

### 状況

再起動後、以下を実行した。

```bash
cat /data/test.txt
```

ファイルがない、または読めない。

### 考えられる原因

```text
/data が追加EBSではなく、通常の空ディレクトリになっている
追加EBSがマウントされていない
test.txtを作った後にmkfsを再実行して消した
別のディスクをマウントしている
```

### 確認

```bash
df -h
lsblk
cat /etc/fstab
```

`/data` が `/dev/nvme1n1` に紐づいているか確認する。

---

## 9. EBS削除前の注意

### 状況

実践後にリソースを削除する。

### 注意

EC2を終了しても、追加EBSが残る場合がある。

### 確認場所

```text
EC2
→ Elastic Block Store
→ ボリューム
```

1GiBの追加EBSが残っていないか確認する。

残っている場合は手動で削除する。

---

## トラブル時の基本確認順

```text
1. lsblkでディスクが見えるか
2. df -hでマウントされているか
3. file -sでファイルシステムがあるか
4. /dataディレクトリがあるか
5. mountできるか
6. blkidでUUIDを確認する
7. /etc/fstabの記述を確認する
8. systemctl daemon-reloadを実行する
9. mount -aで確認する
10. reboot後にdf -hとcatで確認する
```

---

## 今回の一番重要なトラブル理解

```text
lsblkに出ているのにdf -hに出ない
```

これは、

```text
ディスクとしては認識されている
でも、まだLinux上の保存場所としては使えない
```

ということ。

この場合は、フォーマット済みかどうかを確認し、必要に応じてマウントする。
