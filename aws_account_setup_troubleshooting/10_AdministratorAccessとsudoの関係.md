
# 10. AdministratorAccessとsudoの関係がわからなかった
## 状況

toki-admin に AdministratorAccess を付与した。

これがLinuxのsudoに近いものなのか疑問に感じた。

## 原因

AWSの権限管理とLinuxのsudoの違いを理解できていなかった。

## 対応

以下のように整理した。
```txt
toki-admin
= sudoできる一般ユーザーに近い
```
```txt
AdministratorAccess
= AWSサービスやリソースをほぼ操作できる強い権限
```
## 学び

AdministratorAccessは、sudoコマンドそのものではない。

Linuxで言うと、sudoersに入っていて強い操作ができる一般ユーザーに近い。

ただし、AWSではBillingなど一部の機能は別途許可が必要な場合がある。
