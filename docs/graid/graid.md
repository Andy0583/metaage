### **<font color="red">AE Pre-installer on RHEL</font>**
```
LOCAL_ISO_PATH=/mnt/dvd/ \
DKMS_PKG_PATH=/root/dkms-3.4.3-2.el9.noarch.rpm \
bash graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run \
-pl ae --offline-install
```

### **<font color="red">重要 KB </font>**
[NVIDIA Driver更新後，GRAID服務無法使用](kb001.md)<br>
[AI Server安裝八張GPU時，無法偵測到GRAID卡](kb002.md)
