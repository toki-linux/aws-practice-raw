# 使用コマンド集

## EC2接続
ssh -i キー名.pem ubuntu@EC2のパブリックIPv4アドレス

## nginx
sudo apt update
sudo apt install -y nginx
systemctl status nginx

## Webページ作成
echo "Hello from EC2" | sudo tee /var/www/html/index.html
curl localhost

## AWS CLI v2インストール
sudo apt update
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

## IAM Role確認
aws sts get-caller-identity

## S3操作
aws s3 ls
aws s3 ls s3://バケット名
aws s3 cp s3://バケット名/test.txt .
aws s3 cp s3://バケット名/test.txt -

## EBS確認
lsblk
df -h

## 追加EBS初期化とマウント
sudo mkfs -t ext4 /dev/nvme1n1   # デバイス名はlsblkで確認
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
lsblk
df -h

## テストファイル作成
echo "hello ebs" | sudo tee /data/test.txt
cat /data/test.txt

## UUID確認
sudo blkid /dev/nvme1n1
sudo cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
# 追記例
```txt
# UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /data ext4 defaults,nofail 0 2
```
sudo systemctl daemon-reload
sudo mount -a
sudo reboot

## 再起動後確認
lsblk
df -h
cat /data/test.txt

## トラブル時確認
journalctl -xb
