# SSM Session Manager 再構築ログ

## 概要

前回は手順に沿って、NAT GatewayなしでPrivate EC2へSession Manager接続する構成を作成した。

今回は、各リソースの意味を確認しながら、同じ構成を1から再構築した。

## 再構築で意識したこと

- IAM Roleは、EC2内のSSM AgentがSystems Managerと通信するための許可証
- VPC Endpointは、Private EC2からAWS Systems Managerへ届くための専用窓口
- EC2側のInboundはなし
- Endpoint側のSecurity Groupで、Private EC2からのHTTPS 443を許可
- Security Groupはステートフルなので、EC2から開始した通信の戻りはInboundなしでも戻れる
- Private DNSにより、SSM系サービス名をEndpointのPrivate IPへ向ける

## 再構築したリソース

- IAM Role
- VPC
- Private Subnet
- Security Group
- VPC Endpoint
  - ssm
  - ssmmessages
  - ec2messages
- Private EC2

## 確認したこと

Session ManagerでPrivate EC2へ接続できた。

```bash
whoami
hostname
ip a
curl -I https://aws.amazon.com
```
## 結果
whoami で ssm-user を確認
ip a でPrivate IPを確認
curl -I https://aws.amazon.com は失敗
NAT Gateway / Internet GatewayなしでもSession Manager接続は成功
## 学び

今回の構成では、EC2へ外から直接入っているのではなく、EC2内のSSM AgentがSystems Managerと通信できる状態を作ることで、AWSコンソール上のSession ManagerからEC2を操作できる。

つまり、SSH接続とは違い、以下の考え方になる。

SSH:
自分 → EC2

SSM:
自分 → Systems Manager ← EC2
削除

検証後、以下を削除した。
```txt
EC2インスタンス
VPC Endpoint 3つ
Security Group
Subnet
VPC
IAM Role
```
