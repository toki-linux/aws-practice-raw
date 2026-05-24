# screenshots

このディレクトリには、CloudWatch Alarm実践の確認用スクリーンショットを保存します。

## 保存するスクリーンショット例

- EC2インスタンス画面
- CloudWatch MetricsのCPUUtilizationグラフ
- CloudWatch Alarm作成画面
- `ec2-high-cpu-alarm` が `OK` の状態
- `ec2-high-cpu-alarm` が `ALARM` の状態
- CPU負荷低下後に `OK` に戻った状態

---

## 注意点

スクリーンショットをGitHubにアップロードする前に、以下の情報が写っていないか確認します。

- AWSアカウントID
- メールアドレス
- 不要なパブリックIP
- 秘密情報
- 個人情報
- 不要なリソース名

---

## 今回確認したい画面

### CloudWatch Metrics

対象メトリクス：

```text
CPUUtilization
```

### CloudWatch Alarm

アラーム名：

```text
ec2-high-cpu-alarm
```

確認した状態変化：

```text
OK
↓
ALARM
↓
OK
```

---

## スクリーンショットを載せる目的

今回の実践では、CloudWatch Alarmがメトリクスのしきい値をもとに状態変化することを確認しました。

スクリーンショットを残すことで、以下の証跡になります。

```text
CPUUtilizationが上がったこと
ALARM状態になったこと
負荷低下後にOK状態へ戻ったこと
```
