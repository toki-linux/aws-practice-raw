# CloudWatch Alarm実践 トラブルシューティング

## 概要

このファイルでは、CloudWatch Alarm実践で発生した問題や、確認ポイントをまとめます。

---

## 1. CPU負荷をかけてもALARM状態にならない

### 症状

EC2で `stress` を実行してCPU負荷をかけたが、CloudWatch Alarmが `ALARM` 状態になりませんでした。

実行したコマンド：

```bash
stress --cpu 1 --timeout 300
```

最初のアラーム条件：

```text
CPUUtilization >= 50
```

---

## 原因

CloudWatchのメトリクスグラフを確認したところ、CPU使用率が50%に届いていませんでした。

つまり、アラームが壊れていたわけではなく、設定したしきい値を超えていなかったことが原因でした。

```text
条件：CPUUtilization >= 50%
実際：CPUUtilization が50%未満
結果：ALARMにならない
```

---

## 対応

学習用として、しきい値を50%から20%に下げました。

```text
CPUUtilization >= 20
```

その後、再度CPU負荷をかけました。

```bash
stress --cpu 2 --timeout 600
```

結果として、CPUUtilizationが20%を超え、CloudWatch Alarmが `ALARM` 状態になりました。

---

## 学び

CloudWatch Alarmは、EC2で `stress` を実行したこと自体を見ているわけではありません。

CloudWatch上の `CPUUtilization` メトリクスが、設定した条件を満たしているかどうかを判定しています。

---

## 2. CloudWatchの反映が遅い

### 症状

EC2でCPU負荷をかけても、CloudWatchのグラフやアラーム状態にすぐ反映されませんでした。

---

## 原因

CloudWatch MetricsやCloudWatch Alarmは、EC2内の `top` コマンドのように完全なリアルタイム表示ではありません。

メトリクスの反映やアラームの状態変化には、数十秒から数分程度かかることがあります。

---

## 対応

EC2側では `top` を使って、CPU負荷が実際に発生しているか確認します。

```bash
top
```

CloudWatch側では、数分待ってからグラフやアラーム状態を確認します。

---

## 学び

```text
top
→ EC2内でほぼリアルタイムに確認できる

CloudWatch Metrics / Alarm
→ 少し遅れて反映されることがある
```

---

## 3. しきい値の設定が高すぎる

### 症状

CPU負荷をかけても、アラーム状態にならない。

---

## 原因

設定したしきい値が高すぎると、CPU負荷をかけても条件を満たさないことがあります。

今回、最初に設定した50%では、CPUUtilizationがしきい値に届きませんでした。

---

## 対応

学習用で状態変化を確認したい場合は、しきい値を低めに設定します。

例：

```text
CPUUtilization >= 20
```

または、より確実に確認したい場合は以下も検討します。

```text
CPUUtilization >= 10
```

---

## 注意点

実務では、しきい値を低くしすぎると不要なアラームが増える可能性があります。

今回の20%設定は学習用です。

---

## 4. アラーム作成直後にINSUFFICIENT_DATAになる

### 症状

アラーム作成直後に、状態が `INSUFFICIENT_DATA` になりました。

---

## 意味

`INSUFFICIENT_DATA` は、異常ではありません。

CloudWatch Alarmがまだ判定に必要なデータを十分に持っていない状態です。

---

## 対応

しばらく待つと、メトリクスデータが入り、`OK` または `ALARM` に変わります。

---

## 状態の意味

```text
OK
→ 条件を満たしていない

ALARM
→ 条件を満たしている

INSUFFICIENT_DATA
→ 判定するためのデータが足りない
```

---

## 5. 通知やアクションは必要か

### 今回の判断

今回は通知やアクションは設定しませんでした。

```text
通知：なし
アクション：なし
```

---

## 理由

今回の目的は、メール通知や自動復旧ではなく、CloudWatch Alarmの状態変化を確認することだったためです。

```text
OK
↓
ALARM
↓
OK
```

この流れを確認できれば、今回の実践としては成功です。

---

## 6. EC2を削除した後もアラームが残る

### 注意点

EC2インスタンスを終了しても、CloudWatch Alarmが自動で削除されるとは限りません。

---

## 対応

検証後はCloudWatch Alarmも確認して削除します。

```text
CloudWatch
→ アラーム
→ すべてのアラーム
→ 対象アラームを選択
→ 削除
```

---

## 今回削除したアラーム

```text
ec2-high-cpu-alarm
```

通知やアクションを設定していない場合、特別な解除作業は不要です。

---

## まとめ

CloudWatch Alarmが期待通りに動かない場合は、以下の順番で確認します。

```text
1. 対象メトリクスは正しいか
2. 対象EC2のCPUUtilizationを見ているか
3. CloudWatchのグラフでしきい値を超えているか
4. しきい値が高すぎないか
5. データポイントや期間の設定は適切か
6. INSUFFICIENT_DATAではないか
7. CloudWatchの反映待ちではないか
8. stressが実際に動いているか
```

今回の重要な学びは、アラームが鳴らないときに、まずメトリクスグラフを見ることです。
