# SSH接続

## 概要

作成したEC2インスタンスに、秘密鍵を使ってSSH接続した。

今回はUbuntu Serverを使用したため、SSH接続時のLinuxユーザー名は `ubuntu` を使用した。

## IAMユーザーとLinuxユーザーの違い

AWS管理画面にログインするユーザーと、EC2内で操作するユーザーは別物である。

```text
toki-admin
→ AWSマネジメントコンソールにログインするIAMユーザー

ubuntu
→ EC2上のUbuntuサーバにSSH接続するときのLinuxユーザー
```

## 秘密鍵の配置

ダウンロードした秘密鍵ファイルをMacのホームディレクトリに配置した。

例：

```text
~/toki-ec2-key.pem
```

## 秘密鍵の権限変更

SSH接続前に、秘密鍵の権限を変更した。

```bash
chmod 400 toki-ec2-key.pem
```

## SSH接続コマンド

```bash
ssh -i toki-ec2-key.pem ubuntu@パブリックIPv4アドレス
```

例：

```bash
ssh -i toki-ec2-key.pem ubuntu@xx.xx.xx.xx
```

## 初回接続時の確認

初回接続時に以下のような表示が出た。

```text
Are you sure you want to continue connecting?
```

これは、接続先のサーバを信頼するか確認するメッセージである。

`yes` と入力して接続を続行した。

## 接続確認

接続後、以下のコマンドで確認した。

```bash
whoami
hostname
```

`whoami` の結果が以下であれば、Ubuntuサーバにログインできている。

```text
ubuntu
```

## 学んだこと

- EC2へのSSH接続には秘密鍵が必要である
- Ubuntu Serverの初期ユーザーは `ubuntu` である
- IAMユーザー名とEC2内のLinuxユーザー名は別物である
- 秘密鍵の権限が広すぎるとSSH接続時にエラーになる
- 初回接続時には接続先サーバを信頼する確認が表示される
