# EC2 CPU Monitoring with CloudWatch Alarm

## 概要

このリポジトリは、EC2のCPU使用率をCloudWatchで監視し、CloudWatch Alarmでしきい値超過を検知する学習記録です。

EC2上で `stress` コマンドを使って意図的にCPU負荷を発生させ、CloudWatchの `CPUUtilization` メトリクスが上昇することを確認しました。

その後、CloudWatch Alarmを作成し、CPU使用率がしきい値を超えると `ALARM` 状態になり、負荷が下がると `OK` 状態へ戻ることを確認しました。

---

## 目的

この課題の目的は、CloudWatch MetricsとCloudWatch Alarmの基本的な仕組みを理解することです。

- EC2のCPU使用率をCloudWatch Metricsで確認する
- `CPUUtilization` メトリクスを理解する
- CloudWatch Alarmを作成する
- CPU使用率がしきい値を超えたときに `ALARM` 状態になることを確認する
- CPU使用率が下がったときに `OK` 状態へ戻ることを確認する
- アラームが発火しない場合に、メトリクスグラフを見て原因を切り分ける

---

## 構成

```text
AWS
├── EC2
│   └── stressでCPU負荷を発生
│
└── CloudWatch
    ├── Metrics
    │   └── CPUUtilization
    │
    └── Alarm
        └── ec2-high-cpu-alarm
```

---

## 使用したAWSサービス

- EC2
- CloudWatch Metrics
- CloudWatch Alarm
- Security Group

---

## 今回使わなかったもの

今回の実践では、以下は使用しませんでした。

- CloudWatch Agent
- EC2用IAM Role
- SNS通知

理由は、EC2の `CPUUtilization` はAWS側が標準メトリクスとして自動収集しているためです。

CloudWatch Logsのように、EC2内のログファイルをCloudWatchへ送る場合はCloudWatch AgentとIAM Roleが必要ですが、今回のCPU使用率監視では不要でした。

---

## CloudWatch Logsとの違い

前回のCloudWatch Logs実践では、EC2内のnginxログをCloudWatch Logsへ送信しました。

```text
EC2内のnginxログ
↓
CloudWatch Agentが読む
↓
IAM Roleの権限でCloudWatch Logsへ送信
↓
AWSコンソールで確認
```

一方、今回のCloudWatch Alarm実践では、EC2の標準メトリクスであるCPU使用率を使いました。

```text
EC2のCPU使用率
↓
AWS側が標準メトリクスとして収集
↓
CloudWatch Metricsで確認
↓
CloudWatch Alarmが条件判定
```

つまり、違いは以下です。

```text
CloudWatch Logs
→ EC2内のログファイルを見る
→ CloudWatch AgentとIAM Roleが必要

CloudWatch Metrics / Alarm
→ EC2のCPU使用率などの数値を見る
→ 今回はCloudWatch AgentもIAM Roleも不要
```

---

## 実施内容

### 1. EC2インスタンスの作成

UbuntuのEC2インスタンスを1台作成しました。

今回の実践ではCPU使用率を確認するだけなので、特別な設定はほぼ不要です。

設定内容は以下です。

```text
AMI：Ubuntu
インスタンスタイプ：t2.micro または t3.micro
Security Group：SSH 22番を許可
IAM Role：なし
CloudWatch Agent：なし
```

---

### 2. SSH接続

作成したEC2にSSH接続しました。

```bash
ssh -i キー名.pem ubuntu@EC2のパブリックIPv4アドレス
```

---

### 3. stressのインストール

CPU負荷を意図的に発生させるため、`stress` をインストールしました。

```bash
sudo apt update
sudo apt install -y stress
```

`stress` は、CPUやメモリなどに負荷をかけるためのツールです。

---

### 4. CloudWatch Alarmの作成

AWSコンソールでCloudWatchを開き、EC2の `CPUUtilization` を対象にアラームを作成しました。

```text
CloudWatch
→ アラーム
→ すべてのアラーム
→ アラームの作成
```

選択したメトリクスは以下です。

```text
EC2
→ インスタンス別メトリクス
→ 対象EC2の CPUUtilization
```

---

## 最初のアラーム設定

最初は以下の条件でアラームを作成しました。

```text
メトリクス：CPUUtilization
統計：平均
しきい値の種類：静的
条件：CPUUtilization >= 50
データポイント：1/1
欠落データの処理：欠落データを見つかりませんとして処理
通知：なし
アクション：なし
アラーム名：ec2-high-cpu-alarm
```

通知やアクションは設定しませんでした。

今回の目的は通知ではなく、アラーム状態が `OK → ALARM → OK` と変化することを確認することだったためです。

---

## CPU負荷の発生

最初に以下のコマンドでCPU負荷をかけました。

```bash
stress --cpu 1 --timeout 300
```

意味は以下です。

```text
--cpu 1
→ CPUに負荷をかける処理を1つ動かす

--timeout 300
→ 300秒、つまり5分間実行する
```

しかし、CloudWatchのグラフを確認すると、CPU使用率が50%に届いていませんでした。

そのため、アラームは `ALARM` 状態になりませんでした。

---

## 原因の切り分け

アラームが `ALARM` 状態にならなかった理由は、CloudWatch Alarmの故障ではなく、単純に `CPUUtilization` がしきい値に届いていなかったためです。

```text
条件：CPUUtilization >= 50%
実際：CPUUtilization が50%未満
結果：ALARMにならない
```

このとき、CloudWatchのメトリクスグラフを確認することで原因を切り分けました。

---

## しきい値の変更

学習用として、しきい値を50%から20%に変更しました。

```text
CPUUtilization >= 20
```

しきい値を下げた理由は、今回の目的が実務向けの適切なしきい値を決めることではなく、CloudWatch Alarmの状態変化を確認することだったためです。

---

## 再度CPU負荷を発生

しきい値を20%へ変更したあと、再度CPU負荷をかけました。

```bash
stress --cpu 2 --timeout 600
```

意味は以下です。

```text
--cpu 2
→ CPUに負荷をかける処理を2つ動かす

--timeout 600
→ 600秒、つまり10分間実行する
```

その結果、CloudWatchの `CPUUtilization` が20%を超え、CloudWatch Alarmが `ALARM` 状態になりました。

その後、CPU負荷が下がると、アラームは `OK` 状態へ戻りました。

---

## 確認できた流れ

今回確認できた流れは以下です。

```text
EC2でCPU負荷を上げる
↓
CloudWatchのCPUUtilizationが上がる
↓
しきい値20%を超える
↓
CloudWatch AlarmがALARM状態になる
↓
stress終了後、CPU使用率が下がる
↓
CloudWatch AlarmがOK状態に戻る
```

---

## 学び

### CloudWatch Alarmはメトリクスを見て判定する

CloudWatch Alarmは、EC2内で `stress` を実行したかどうかを直接見ているわけではありません。

CloudWatch上のメトリクスである `CPUUtilization` を見て、設定した条件を満たすかどうかを判定しています。

```text
CPUUtilization がしきい値を超えた
→ ALARM

CPUUtilization がしきい値を下回った
→ OK
```

---

### アラームにならないときはメトリクスグラフを見る

アラーム状態にならない場合は、まずCloudWatchのメトリクスグラフを見ることが重要です。

今回も、最初はアラームが発火しませんでしたが、グラフを見ることでCPU使用率が50%に届いていないことを確認できました。

---

### CloudWatchは完全なリアルタイムではない

EC2内の `top` コマンドは、ほぼリアルタイムにCPU使用状況を確認できます。

一方、CloudWatch MetricsやCloudWatch Alarmは、反映まで少し時間がかかることがあります。

```text
top
→ EC2内でほぼリアルタイムに確認

CloudWatch Metrics / Alarm
→ 数十秒〜数分遅れて反映されることがある
```

---

### 通知・アクションなしでもアラームは作成できる

今回はSNS通知や自動アクションを設定しませんでした。

それでも、CloudWatch Alarmの状態変化は確認できます。

```text
通知なし
アクションなし
状態変化のみ確認
```

学習目的では、この設定で十分です。

---

### CloudWatch AlarmはEC2削除後も確認が必要

EC2インスタンスを終了しても、CloudWatch Alarmが自動で削除されるとは限りません。

検証後は、CloudWatch Alarmを別途確認し、不要であれば削除します。

---

## 削除チェック

検証後、課金防止と整理のため、以下を削除・確認しました。

- EC2インスタンスの終了
- CloudWatch Alarmの削除
- 不要なEBSボリュームが残っていないか確認
- 不要なSecurity Groupの確認
- Key Pairの確認

今回、通知やアクションは設定していなかったため、SNSトピックなどの追加削除は不要でした。

---

## 30秒説明

EC2のCPU使用率をCloudWatchの `CPUUtilization` メトリクスで確認し、一定以上になったら `ALARM` 状態になるCloudWatch Alarmを作成しました。

最初はしきい値を50%に設定しましたが、`stress` 実行後もCPU使用率が50%に届かなかったため、アラーム状態にはなりませんでした。

CloudWatchのメトリクスグラフを確認して原因を切り分け、学習用にしきい値を20%へ変更しました。

再度CPU負荷をかけたところ、`CPUUtilization` が20%を超えて `ALARM` 状態になり、負荷が下がった後に `OK` 状態へ戻ることも確認しました。

---

## まとめ

今回の実践では、CloudWatch MetricsとCloudWatch Alarmを使って、EC2のCPU使用率を監視しました。

特に、アラームが鳴らない場合にメトリクスグラフを確認し、しきい値に達していないことを切り分けられた点が大きな学びでした。

これにより、CloudWatch Alarmは単に作成するだけでなく、メトリクスの値と条件を正しく確認することが重要だと理解できました。
