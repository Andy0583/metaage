**設定Ubuntu**
```
sudo passwd root

sudo ufw disable

sudo vim /etc/ssh/sshd_config
  PermitRootLogin yes

sudo systemctl restart ssh

apt update

apt install corosync-qnetd -y

systemctl status corosync-qnetd
```

**設定PVE**
```
# 建立 SSH 信任
ssh-copy-id root@172.30.58.23

# 在 PVE 叢集中新增 QDevice
pvecm qdevice setup 172.30.58.23

# 驗證
pvecm status
```
