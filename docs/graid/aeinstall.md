### **<font color="red">RHEL9 離線安裝AE (T400)</font>**   
**RHEL設定**
```
# 關閉安全性及防火牆
sed -i 's/enforcing/disabled/' /etc/selinux/config
systemctl stop firewalld &&systemctl disable firewalld

# Mount ISO
mkdir -p /mnt/dvd
mount -o loop /root/rhel-9.6-x86_64-dvd.iso /mnt/dvd

# 修改repo
vi /etc/yum.repos.d/local-dvd.repo
[local-baseos]
name=Red Hat Enterprise Linux BaseOS - Local DVD
baseurl=file:///mnt/dvd/BaseOS
enabled=1
gpgcheck=0

[local-appstream]
name=Red Hat Enterprise Linux AppStream - Local DVD
baseurl=file:///mnt/dvd/AppStream
enabled=1
gpgcheck=0

vi /etc/yum/pluginconf.d/subscription-manager.conf
[main]
enabled=0

dnf clean all
dnf makecache
```

**安裝graid-sr-pre-installer**
[檔案下載](https://dtimis-my.sharepoint.com/:u:/g/personal/andyhsu_ginnet_com_tw/IQBoIKC8omrAQpF1BdAAQrxxAb3Ho2LzFpp4_BqMLyeFuik?e=Q7an7A)
```
LOCAL_ISO_PATH=/mnt/dvd/ DKMS_PKG_PATH=/root/dkms-3.4.3-2.el9.noarch.rpm \
bash graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run \
-pl ae --offline-install
```

**加入T400卡id**
```
vim /etc/graid_pre_installer.conf
EXPECTED_GPU_CARDS="1ff2"
```

**安裝SupremeRAID AE Only for T400**
[檔案下載](https://dtimis-my.sharepoint.com/:u:/g/personal/andyhsu_ginnet_com_tw/IQARKVAs7fgjQaN7ZHb_OfV2AYxJ4dRmUSVmpUtGn9jlMZk?e=sshv5R)
```
bash graid-sr-ae-installer-2.0.0-tu75-193-174.run 
```

### **<font color="red">Ubuntu 線上安裝AE</font>**   
**安裝graid-sr-pre-installer**
```
wget https://download.graidtech.com/driver/pre-install/graid-sr-pre-installer-2.0.x-nv580-288-x86_64.run
chmod +x graid-sr-pre-installer-2.0.x-nv580-288-x86_64.run
bash graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run -pl ae --yes
 # 自動重新開機
```

**安裝graid-sr-installer**
```
wget https://download.graidtech.com/driver/sr-ae/linux/2.0.0-217/release/graid-sr-ae-installer-2.0.0-am86-217-201.run
bash graid-sr-ae-installer-2.0.0-am86-217-201.run --accept-license
```

### **<font color="red">Ubuntu 離線安裝AE (T400)</font>**   
**升級Kernel**
[檔案下載](https://dtimis-my.sharepoint.com/:u:/g/personal/andyhsu_ginnet_com_tw/IQBeTbQAYhQaRKmG5tbCWPijAXjz8T7m5lLUqf7Fgj9lM1I?e=GbNiNC)
```
tar xzvf kernel-6.8.0-138.tar.gz
dpkg -i *.deb
apt-get install -f
update-grub
reboot
```

**安裝所需套件**
[檔案下載](https://dtimis-my.sharepoint.com/:u:/g/personal/andyhsu_ginnet_com_tw/IQA3g6s5SNRrRaVZIqZctW3rAXFVl9QFpzkF0mbO-EQgykE?e=ZVmL9Q)
```
tar xzf graid-complete.tar.gz
dpkg -i *.deb
apt-get install -f
```

**安裝graid-sr-pre-installer**
```
chmod +x graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run

bash graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run \
-pl ae --offline-install --yes

# 自動重新開機
```

**加T400 id**
```
vim /etc/graid_pre_installer.conf
EXPECTED_GPU_CARDS="1ff2"
```

**安裝SupremeRAID AE Only for T400**
[檔案下載](https://dtimis-my.sharepoint.com/:u:/g/personal/andyhsu_ginnet_com_tw/IQARKVAs7fgjQaN7ZHb_OfV2AYxJ4dRmUSVmpUtGn9jlMZk?e=sshv5R)
```
bash graid-sr-ae-installer-2.0.0-tu75-193-174.run --accept-license
```
