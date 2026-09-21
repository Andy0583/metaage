### **<font color="red">內建etcd backup</font>**
**Backup**
```
# SNO：進行etcd備份
ssh core@172.22.46.232

sudo -i

/usr/local/bin/cluster-backup.sh /home/core/assets/backup

chmod 644 /home/core/assets/backup/*

exit

# Bastion：將備份檔案傳出
scp core@172.22.46.232:/home/core/assets/backup/* /root/backup

ls -la /root/backup
```

**Restore**
```
# Bastion：將備份檔案傳入
scp /root/backup/* core@172.22.46.232:/home/core/assets/backup/

# SNO：進行etcd還原
ssh core@172.22.46.232

sudo -i

mv /etc/kubernetes/manifests/etcd-pod.yaml /tmp/manifests-backup-etcd.yaml

mv /etc/kubernetes/manifests/kube-apiserver-pod.yaml /tmp/manifests-backup-apiserver.yaml

mkdir -p /home/core/assets/restore

cp /home/core/assets/backup/* /home/core/assets/restore/

/usr/local/bin/cluster-restore.sh /home/core/assets/restore

ls -la /etc/kubernetes/static-pod-resources/ | grep kube-apiserver-pod

cp /etc/kubernetes/static-pod-resources/kube-apiserver-pod-6/kube-apiserver-pod.yaml /etc/kubernetes/manifests/

systemctl daemon-reload

systemctl restart kubelet

crictl ps | grep -E 'etcd|kube-apiserver'

rm -f /tmp/manifests-backup-etcd.yaml /tmp/manifests-backup-apiserver.yaml

# # Bastion：觸發新的 etcd revision
export KUBECONFIG=/home/ocp-offline/sno-install/auth/kubeconfig
oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$(date --rfc-3339=ns)"'"}}' --type=merge
```
