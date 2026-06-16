
# AWSコスト最適化：Reserved InstancesとCompute Savings Plans

## 概要

AWSでは、長期間使うことが分かっているコンピューティングリソースに対して、割引を受ける方法がある。

代表的なものは以下。

```text
EC2 Reserved Instances
Compute Savings Plans
```

一言で整理すると、以下の違い。

```text
Reserved Instances
→ 主にEC2を安くする

Compute Savings Plans
→ EC2・Fargate・LambdaなどのCompute全体を安くする
```

---

# EC2 Reserved Instancesとは？

## 概要

Reserved Instances、略してRIは、EC2を長期間使う前提で割引を受ける仕組み。

```text
Reserved Instances
=
EC2を長期利用する前提で割引を受ける仕組み
```

通常のオンデマンド料金は、使った分だけ支払う。

RIは、一定期間使うことを前提にすることで、通常より安く利用できる。

---

## 覚え方

```text
Reserved
→ 予約する

Reserved Instances
→ EC2の利用を予約して安くする
```

---

# RIの主な種類

Cloud Practitionerでは、まず以下の2種類を覚えればよい。

```text
Standard RI
Convertible RI
```

---

# Standard RIとは？

## 概要

Standard RIは、割引率が高い代わりに、変更の自由度が低いRI。

```text
Standard RI
=
割引大・柔軟性低め
```

---

## 例え

Standard RIは、特定の路線・区間の定期券に近い。

```text
この路線
この区間
この期間

を長く使うなら安い
```

決まった使い方を長く続ける場合に向いている。

---

## 特徴

```text
割引率が高い
ただし変更の自由度は低い
安定して同じEC2構成を使う場合に向く
```

---

# Convertible RIとは？

## 概要

Convertible RIは、Standard RIより割引率は少し低めだが、変更しやすいRI。

```text
Convertible RI
=
割引やや低め・柔軟性高め
```

---

## 例え

Convertible RIは、条件変更できる定期券に近い。

```text
今はこの路線を使う
でも将来、別の条件に変えるかもしれない
```

このような場合に向いている。

---

## 例

今は以下のEC2を使っている。

```text
t3.micro
```

将来、以下のようなEC2へ変わる可能性がある。

```text
m7i.large
```

Convertible RIなら、条件を満たす範囲でRIを交換できる。

---

# Standard RIとConvertible RIの違い

| 種類 | 割引率 | 柔軟性 | 覚え方 |
|---|---|---|---|
| Standard RI | 高い | 低い | 固定の定期券 |
| Convertible RI | Standardより低め | 高い | 条件変更できる定期券 |

---

## 一言で覚える

```text
Standard RI
→ 割引大・柔軟性低い

Convertible RI
→ 割引やや低め・柔軟性高い
```

---

# Compute Savings Plansとは？

## 概要

Compute Savings Plansは、EC2、Fargate、Lambdaなどのコンピューティング料金を安くするための割引プラン。

```text
Compute Savings Plans
=
Computeを長く使う約束をして安くするプラン
```

---

## 基本の考え方

一定期間、一定額のCompute利用を約束する。

```text
1年または3年
↓
毎時間これくらい使いますと約束
↓
対象のCompute料金が割引される
```

---

## 対象

Compute Savings Plansは、以下に適用される。

```text
EC2
Fargate
Lambda
```

---

## 例え

通常のオンデマンド料金は、ジムの都度払い。

```text
使った分だけ払う
```

Compute Savings Plansは、ジムの月額会員に近い。

```text
継続利用を約束する
↓
その代わり単価が安くなる
```

---

# Compute Savings PlansとEC2 Reserved Instancesの違い

## 結論

```text
Reserved Instances
→ 特定のEC2利用を安くする

Compute Savings Plans
→ EC2・Fargate・LambdaなどのCompute利用を広く安くする
```

---

## 比較表

| 項目 | EC2 Reserved Instances | Compute Savings Plans |
|---|---|---|
| 主な対象 | EC2 | EC2・Fargate・Lambda |
| 柔軟性 | 条件による | 高い |
| 考え方 | EC2の利用条件を予約する | Compute利用額をコミットする |
| 向いている場面 | EC2構成が長期間ほぼ固定 | Compute全体の使い方が変わる可能性がある |

---

# Convertible RIとCompute Savings Plansの違い

## なぜ混ざるのか？

Convertible RIも、Compute Savings Plansも、どちらも柔軟性がある。

```text
Convertible RI
→ 柔軟

Compute Savings Plans
→ 柔軟
```

そのため、最初は混ざりやすい。

---

## 結論

```text
Convertible RI
→ EC2の世界の中で柔軟

Compute Savings Plans
→ EC2以外も含めて柔軟
```

---

## Convertible RI

Convertible RIは、基本的にはEC2の割引。

EC2の利用条件が変わる可能性がある場合に使える。

```text
今
→ t3.micro

将来
→ m7i.large
```

このような変更に対応しやすい。

ただし、対象は基本的にEC2。

---

## Compute Savings Plans

Compute Savings Plansは、対象が広い。

```text
今日
→ EC2を使う

来年
→ Lambdaへ移行する

さらに
→ Fargateも使う
```

このようにComputeサービスの使い方が変わっても、割引を受けやすい。

---

## 比較表

| 項目 | Convertible RI | Compute Savings Plans |
|---|---|---|
| 主な対象 | EC2 | EC2・Lambda・Fargate |
| 柔軟性 | EC2内で高い | Compute全体で高い |
| 考え方 | RIを交換できる | Compute利用額に自動適用される |
| 試験での重要度 | 中 | 高 |

---

# 柔軟性の順番

柔軟性は、次の順番で上がると覚える。

```text
Standard RI
↓
Convertible RI
↓
Compute Savings Plans
```

---

## 覚え方

```text
Standard RI
→ このEC2を長く使うなら安い

Convertible RI
→ EC2の中で条件変更しやすい

Compute Savings Plans
→ EC2・Lambda・Fargateまで含めて柔軟
```

---

# 試験向けまとめ

```text
Reserved Instances
→ EC2の長期利用割引

Standard RI
→ 割引大・柔軟性低い

Convertible RI
→ 割引やや低め・柔軟性高い
→ EC2の中で柔軟

Compute Savings Plans
→ EC2・Fargate・LambdaのCompute料金を割引
→ RIより柔軟
→ 長期的にCompute利用が見込める場合に向く
```


