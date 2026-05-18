## 起きたこと
```txt
toki-adminでAWSにログインした
↓
EC2でUbuntu Serverのインスタンスを作った
↓
そのUbuntu Serverには初期ユーザー ubuntu が用意されていた
↓
秘密鍵を使って ubuntu ユーザーとしてSSH接続した
```
## イメージ
```TXT
toki-admin
＝ AWSの管理者カード

Ubuntu Server
＝ 借りた部屋

ubuntu
＝ その部屋に最初から用意されている初期ユーザー

秘密鍵.pem
＝ その部屋に入るための鍵
```
だから、
```TXT
AWSで使う名前
```
と
```TXT
サーバの中で使う名前
```
は別。

## 学び
AWSコンソールにログインするIAMユーザーと、EC2インスタンス内で操作するLinuxユーザーは別物である。

今回、AWSコンソールにはIAMユーザー `toki-admin` でログインしたが、Ubuntu ServerのEC2インスタンスへSSH接続する際は、AMIにあらかじめ用意されている初期ユーザー `ubuntu` を使用した。

EC2作成時にIAMユーザー名がインスタンス内のLinuxユーザーとして自動作成されるわけではない。
