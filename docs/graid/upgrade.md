### **<font color="red">NVIDIA Driver更新後，GRAID服務無法使用</font>** 
### Issue
**將 NVIDIA 驅動程式從 570.124.04 升級到 580.65.06 後，graid 服務無法啟動。**
```
modprobe: ERROR: could not insert 'graid_nvidia': Invalid argument
graid.service: Failed with result 'exit-code'.
```

### Resolution
**變更 graid_server_pre.sh**
```
cat << 'EOF' >> /usr/bin/graid_server_pre.sh

if ! modprobe graid-nvidia 2>/dev/null; then

    versions=$(dkms status graid | grep -oP 'graid/\K[^,]+' | sort -u)

    for version in $versions; do

        dkms remove graid/$version --all

        dkms install graid/$version

    done

    modprobe graid-nvidia

fi
EOF
```
**重啟服務**
```
systemctl restart graid
systemctl status graid
```
**確認模組載入成功**
```
lsmod | grep graid
```
