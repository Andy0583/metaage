### **<font color="red">GRAID技術公告</font>**

**Deploy container on AE**
```
vllm:
    ipc: host
    environment:
      CUDA_MPS_PIPE_DIRECTORY: /tmp/nvidia-mps
      CUDA_MPS_LOG_DIRECTORY: /tmp/nvidia-log
      LMCACHE_CONFIG_FILE: /workspace/lmcache_config/local-fs.yam
    volumes:
      - /tmp/nvidia-mps:/tmp/nvidia-mps
      - /tmp/nvidia-log:/tmp/nvidia-log
      - /mnt/graid/models:/workspace:ro
      - /mnt/graid/lmcache:/lmcache
```
```
mkdir -p /mnt/graid/models/lmcache_config

cat > /mnt/graid/models/lmcache_config/local-fs.yaml << 'EOF'
chunk_size: 256
local_disk: file:///lmcache
max_local_disk_size: 64
extra_config: {'use_odirect': True}
EOF
```
```
ps-probe:
    ipc: host
    user: "10001:10001"
    environment:
      CUDA_MPS_PIPE_DIRECTORY: /tmp/nvidia-mps
      CUDA_MPS_LOG_DIRECTORY: /tmp/nvidia-log
    volumes:
      - /tmp/nvidia-mps:/tmp/nvidia-mps
      - /tmp/nvidia-log:/tmp/nvidia-log
```
**AE Pre-installer on RHEL**
```
LOCAL_ISO_PATH=/mnt/dvd/ \
DKMS_PKG_PATH=/root/dkms-3.4.3-2.el9.noarch.rpm \
bash graid-sr-pre-installer-2.0.0-nv580-270-x86_64.run \
-pl ae --offline-install
```

