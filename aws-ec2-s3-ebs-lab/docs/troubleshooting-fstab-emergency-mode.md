# /etc/fstab設定ミスによるemergency mode対応

## 概要

追加EBSを /data に自動マウントするために /etc/fstab を編集後、
再起動するとSSH接続不可になり、emergency mode に入った。

---

## 発生症状

- SSH接続できない
- 22番ポートが閉じているように見える
- AWSコンソールのシステムログに emergency mode 表示

---

## 原因

- /etc/fstab に追加EBSの設定を追加
- 起動時に /data のマウントに失敗
- OSが正常起動できず、sshdも起動しない

---

## 対応

1. 別の正常EC2にルートEBSを一時アタッチ
2. /etc/fstab を編集し該当行をコメントアウト
3. `sudo systemctl daemon-reload`
4. `sudo mount -a` でエラーなし確認
5. 元EC2に戻して再起動
6. SSH接続可能になったことを確認

---

## 学び

- fstabは起動時マウント設定。誤記は emergency mode につながる
- UUIDを使うとデバイス名変化に強い
- `nofail`を付けると起動時にマウント失敗してもOS停止を避けられる
- 編集後は必ず `mount -a` で確認
