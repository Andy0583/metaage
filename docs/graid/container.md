### **<font color="red">Deploy container on AE</font>**
**vllm MPS**
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
**KV offload**
```
mkdir -p /mnt/graid/models/lmcache_config

cat > /mnt/graid/models/lmcache_config/local-fs.yaml << 'EOF'
chunk_size: 256
local_disk: file:///lmcache
max_local_disk_size: 64
extra_config: {'use_odirect': True}
EOF
```
**container MPS**
```
ps-probe:
    ipc: host
    environment:
      CUDA_MPS_PIPE_DIRECTORY: /tmp/nvidia-mps
      CUDA_MPS_LOG_DIRECTORY: /tmp/nvidia-log
    volumes:
      - /tmp/nvidia-mps:/tmp/nvidia-mps
      - /tmp/nvidia-log:/tmp/nvidia-log
```
