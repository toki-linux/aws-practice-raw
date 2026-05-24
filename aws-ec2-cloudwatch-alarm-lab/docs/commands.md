# CloudWatch Alarm実践 使用コマンド集

## 概要

このファイルでは、EC2のCPU使用率をCloudWatch Alarmで監視する実践で使用したコマンドを整理します。

今回の実践では、EC2にCPU負荷をかけるために `stress` を使用しました。

---

## 1. パッケージ一覧を更新する

```bash
sudo apt update
```

## 意味

Ubuntuが参照するパッケージ一覧を最新化します。

ソフトウェアをインストールする前に実行します。

---

## 2. stressをインストールする

```bash
sudo apt install -y stress
```

## 意味

CPUやメモリなどに負荷をかけるためのツールである `stress` をインストールします。

## 分解

```text
sudo
→ 管理者権限で実行する

apt install
→ パッケージをインストールする

-y
→ 確認メッセージに自動で yes と答える

stress
→ インストールするパッケージ名
```

---

## 3. CPUに負荷をかける

```bash
stress --cpu 1 --timeout 300
```

## 意味

CPUに負荷をかける処理を1つ動かし、300秒間実行します。

## 分解

```text
stress
→ 負荷をかけるコマンド

--cpu 1
→ CPUに負荷をかけるワーカーを1つ動かす

--timeout 300
→ 300秒間実行する
```

---

## 4. より強めにCPU負荷をかける

```bash
stress --cpu 2 --timeout 600
```

## 意味

CPUに負荷をかける処理を2つ動かし、600秒間実行します。

今回、最初の `stress --cpu 1 --timeout 300` ではCPU使用率が50%に届かなかったため、より強めに負荷をかける目的で使用しました。

## 分解

```text
--cpu 2
→ CPUに負荷をかけるワーカーを2つ動かす

--timeout 600
→ 600秒、つまり10分間実行する
```

---

## 5. EC2内でCPU状態を確認する

```bash
top
```

## 意味

EC2内でCPU使用率や動作中のプロセスを確認します。

CloudWatchよりもリアルタイムに近い状態で確認できます。

## 見るポイント

```text
stress プロセスが動いているか
CPU使用率が上がっているか
```

終了する場合は、`q` キーを押します。

---

## 6. stressが動いているか確認する

```bash
ps aux | grep stress
```

## 意味

実行中のプロセス一覧から `stress` を探します。

## 分解

```text
ps aux
→ 実行中のプロセス一覧を表示する

grep stress
→ stress という文字を含む行だけ表示する
```

---

## 7. stressを途中で止める場合

```bash
pkill stress
```

## 意味

実行中の `stress` プロセスを終了します。

通常は `--timeout` で指定した時間が過ぎると自動終了しますが、途中で止めたい場合に使用します。

---

## 今回の最小コマンドセット

```bash
sudo apt update
sudo apt install -y stress

stress --cpu 1 --timeout 300
stress --cpu 2 --timeout 600

top
```

---

## コマンドの役割まとめ

```text
apt
→ stressをインストールする

stress
→ CPU負荷を発生させる

top
→ EC2内でCPU使用状況を確認する

ps aux | grep stress
→ stressが動いているか確認する

pkill stress
→ stressを途中停止する
```
