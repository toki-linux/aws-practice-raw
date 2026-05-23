iamroleをs３readonlyで作成しインスタンスに入れた。
インスタンスを作成しnginxを起動しWebページを作成した
s３を作成し中にtest.txtを入れた。
インスタンスでs３の中身が見えるか確認
aws --versionでawscliが使えるかチェックしたが使える状態ではなかった。
sudo apt install awscliで入れようとしたがパッケージがないと言われた。
なので、直接ダウンロードすることにした。
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
このときzipもなかったのでインストールした。これはできた。
awscliが使える状態になったので、s３のtest.txtを見に行くことにした
aws s3 ls
aws s3 ls s3://バケット名
aws s3 cp s3://バケット名/test.txt .
カレンとディレクトリにtest.txtをコピー
cat test.txtでチェック
s３のtest.txtの内容見れた
追加ボリュームを作成した。EBS
リージョンを間違えないこととボリューム名は/dev/sdを選択する
アタッチできたか確認lsblk
df -h では確認できなかった。つまり、アタッチはできているが、マウントはできていない。つまり、まだボリュームとして使用できない
/data/ディレクトリを作成し、これの配下にマウントする
その前にファイルシステムを作成する
sudo mkfs -t ext4 /dev/nvme1n1
これでただのディスクだったものが使えるファイルシステムになった
やっとマウント
mount /dev/nvme1n1 /data

