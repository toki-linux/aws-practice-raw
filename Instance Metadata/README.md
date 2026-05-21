今回の理解としてはこれでOK
Instance Metadata
= EC2自身に関する情報

IMDS
= その情報をEC2内から取得する仕組み

IMDSv2
= トークンを使って安全に取得する方式

169.254.169.254
= EC2内からInstance Metadataへアクセスするための特別な窓口

一言でまとめるなら、

Instance Metadata / IMDSv2 は、EC2が自分自身の情報を取得する機能。
169.254.169.254 は、その情報を取りに行くためのEC2内部専用の問い合わせ先。
