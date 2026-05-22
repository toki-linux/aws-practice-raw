# EBSアタッチ後のUbuntu側作業フロー

## 目的

EC2にアタッチした追加EBSを、Ubuntu上で `/data` として使えるようにする。

今回の流れは以下。

```text
追加EBSが認識されているか確認する
↓
ファイルシステムを作成する
↓
/data ディレクトリを作る
↓
追加EBSを /data にマウントする
↓
ファイルを作成して確認する
↓
/etc/fstab に設定する
↓
再起動後も自動マウントされるか確認する
```

---

## 1. SSH接続する

EC2インスタンスへSSH接続する。

```bash
ssh -i キー名.pem ubuntu@パブリックIP
```

---

## 2. 追加EBSが認識されているか確認する

まず、Ubuntu側でディスク一覧を確認した。

```bash
lsblk
```

実行結果の例。

```text
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0          7:0    0 28.2M  1 loop /snap/amazon-ssm-agent/13009
loop1          7:1    0   74M  1 loop /snap/core22/2411
loop2          7:2    0 49.3M  1 loop /snap/snapd/26865
nvme0n1      259:0    0    8G  0 disk 
├─nvme0n1p1  259:1    0    7G  0 part /
├─nvme0n1p14 259:2    0    4M  0 part 
├─nvme0n1p15 259:3    0  106M  0 part /boot/efi
└─nvme0n1p16 259:4    0  913M  0 part /boot
nvme1n1      259:5    0    1G  0 disk 
```

この結果では、追加EBSは以下。

```text
nvme1n1
```

理由。

```text
サイズが1G
マウントポイントがない
パーティションがない
今回追加したEBSのサイズと一致している
```

---

## 3. lsblkとdf -hの違い

今回の理解で重要なのは、`lsblk` と `df -h` の違い。

```text
lsblk
→ Linuxが認識しているディスクやパーティションを見る

df -h
→ 実際にマウントされ、ファイル保存場所として使えるものを見る
```

つまり、以下の状態は正常。

```text
lsblkでは nvme1n1 が見える
df -hでは /data が見えない
```

これは、

```text
追加EBSはEC2にアタッチされている
でも、まだマウントされていない
```

という状態。

---

## 4. ファイルシステムの有無を確認する

追加EBSにファイルシステムがあるか確認した。

```bash
sudo file -s /dev/nvme1n1
```

まだファイルシステムがない場合の例。

```text
/dev/nvme1n1: data
```

これは、まだUbuntuがファイルを保存・管理できる形式になっていない状態。

---

## 5. ファイルシステムを作成する

追加EBSをext4形式で使えるようにした。

```bash
sudo mkfs -t ext4 /dev/nvme1n1
```

このコマンドの意味。

```text
/dev/nvme1n1 というディスクを
ext4 というファイルシステムで初期化する
```

注意点。

```text
mkfsはディスクを初期化するコマンド
対象を間違えるとデータが消える
ルートEBSではなく、追加EBSを指定する
```

---

## 6. ファイルシステムを作る意味

EBSをアタッチしただけでは、Ubuntuから見ると新しい空のディスクが接続された状態。

そのままだと、Ubuntuはファイルやディレクトリをどう保存するか分からない。

そこで、ファイルシステムを作成する。

```text
ファイルシステムを作る
= Ubuntuがそのディスクにファイルやディレクトリを保存できる形式にする
```

今回使ったファイルシステム。

```text
ext4
```

---

## 7. マウント先ディレクトリを作成する

追加EBSを接続する場所として、`/data` ディレクトリを作成した。

```bash
sudo mkdir /data
```

`/data` は、追加EBSを使うための入口になる。

```text
追加EBS
→ /data にマウント
→ /data 配下にファイルを保存できる
```

---

## 8. 追加EBSを /data にマウントする

追加EBSを `/data` にマウントした。

```bash
sudo mount /dev/nvme1n1 /data
```

このコマンドの意味。

```text
/dev/nvme1n1 を /data に接続する
```

これで、Ubuntu上では `/data` が追加EBSの保存場所になる。

---

## 9. マウントできたか確認する

マウント後、`df -h` で確認した。

```bash
df -h
```

`/data` が表示されれば、マウント成功。

また、`lsblk` でも確認できる。

```bash
lsblk
```

期待する状態。

```text
nvme1n1  1G  disk  /data
```

---

## 10. /data にファイルを作成する

追加EBS上にファイルを書き込めるか確認した。

```bash
echo "hello from additional ebs" | sudo tee /data/test.txt
```

中身を確認した。

```bash
cat /data/test.txt
```

表示例。

```text
hello from additional ebs
```

ここまでで、追加EBSを手動マウントして使える状態になった。

---

## 11. 自動マウントのためにUUIDを確認する

再起動後も自動で `/data` にマウントされるようにするため、UUIDを確認した。

```bash
sudo blkid /dev/nvme1n1
```

表示例。

```text
/dev/nvme1n1: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" BLOCK_SIZE="4096" TYPE="ext4"
```

`UUID="..."` の値を `/etc/fstab` に使う。

---

## 12. /etc/fstabをバックアップする

自動マウント設定を変更する前に、`/etc/fstab` をバックアップした。

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

`/etc/fstab` は起動時のマウント設定に関わる重要ファイルなので、編集前にバックアップしておく。

---

## 13. /etc/fstabを編集する

`/etc/fstab` を開く。

```bash
sudo nano /etc/fstab
```

一番下に以下の形式で追記する。

```text
UUID=自分のUUID /data ext4 defaults,nofail 0 2
```

例。

```text
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults,nofail 0 2
```

---

## 14. nofailを付けた理由

`nofail` を付けることで、もし追加EBSが見つからない場合でも、Ubuntuの起動失敗を防ぎやすくなる。

```text
nofail
→ 対象のディスクがなくても起動を止めない設定
```

練習環境では、安全のために付けておく。

---

## 15. fstabの設定を反映・確認する

`/etc/fstab` を編集した後、systemdに再読み込みさせた。

```bash
sudo systemctl daemon-reload
```

その後、fstabの内容でマウントできるか確認した。

```bash
sudo mount -a
```

何も表示されなければ基本的にOK。

確認。

```bash
df -h
lsblk
cat /data/test.txt
```

---

## 16. systemctl daemon-reloadが必要だった理由

`/etc/fstab` を変更した後、以下のような表示が出た。

```text
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

これは、`/etc/fstab` を変更したが、systemdがまだ古い内容を使っているという意味。

そのため、以下を実行した。

```bash
sudo systemctl daemon-reload
```

その後、再度確認した。

```bash
sudo mount -a
```

---

## 17. 再起動して確認する

自動マウントできるか確認するため、EC2を再起動した。

```bash
sudo reboot
```

再起動後、再度SSH接続した。

```bash
ssh -i キー名.pem ubuntu@パブリックIP
```

確認。

```bash
lsblk
df -h
cat /data/test.txt
```

`/data` がマウントされており、`test.txt` の中身が読めれば成功。

---

## 18. 一度シャットダウン後に確認した状態

一度シャットダウンした後、再確認した。

そのときの状態。

```text
lsblkでは nvme1n1 が存在している
df -hでは /data が表示されない
```

これは以下の状態。

```text
追加EBSはEC2にアタッチされている
しかし、まだ /data にマウントされていない
```

この場合、1から全部やり直す必要はない。

すでにファイルシステム作成済みなら、`mkfs` は実行しない。

```text
mkfsを再実行すると、追加EBS内のデータが消える
```

---

## 19. 最終的に確認できたこと

```text
追加EBSをEC2にアタッチできた
Ubuntuで /dev/nvme1n1 として認識できた
ext4ファイルシステムを作成できた
/data にマウントできた
/data/test.txt を作成できた
/etc/fstab に自動マウント設定を書けた
再起動後も /data がマウントされた
再起動後も /data/test.txt の中身を確認できた
```

---

## 今回の重要ポイント

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
lsblkにあるのにdf -hにない
→ ディスクは認識されているが、まだマウントされていない
```
