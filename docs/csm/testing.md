**設定iSCSI及Multipath**
```
cat > 99z-master-zzz-iscsi-config.yaml << 'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99z-master-zzz-iscsi-config
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - name: iscsid.service
        enabled: true
      - name: multipathd.service
        enabled: true
EOF

oc apply -f 99z-worker-zzz-iscsi-config.yaml

oc get mcp master -w

# SNO會重啟（5分鐘）

oc debug node/sno -- chroot /host systemctl is-enabled iscsid multipathd
```

**建立Storage Class**
```
cat > sc-nas.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: "ps-nas"
provisioner: "csi-powerstore.dellemc.com"
parameters:
  arrayID: "PS3ca9ddfd760c"
  csi.storage.k8s.io/fstype: "nfs"
  nasName: "andy-nas24"
  allowRoot: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF

oc create -f sc-nas.yaml

cat > sc-iscsi.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ps-iscsi
provisioner: csi-powerstore.dellemc.com
parameters:
  arrayId: "PS3ca9ddfd760c"
  protocol: "iSCSI"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF

oc create -f sc-iscsi.yaml
```

**建立PVC**
```
oc create ns test

cat > pvc-nas.yaml << 'EOF'
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-nas-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 11Gi
  storageClassName: ps-nas
EOF

oc create -f pvc-nas.yaml

cat > pvc-iscsi.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-iscsi-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ps-iscsi
  resources:
    requests:
      storage: 15Gi
EOF

oc create -f pvc-iscsi.yaml
```

**建立Pod**
```
cat > pod-iscsi.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: csi-iscsi-test-pod
  namespace: test
  labels:
    app: csi-iscsi-test
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: test-container
      image: harbor.ocp.andy.com/dell/csi-powerstore:v2.17.0
      command: ["/bin/sh", "-c", "sleep infinity"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        runAsNonRoot: true
        runAsUser: 1000670000
      volumeMounts:
        - name: iscsi-test-volume
          mountPath: /data
  volumes:
    - name: iscsi-test-volume
      persistentVolumeClaim:
        claimName: test-iscsi-pvc
EOF

oc create -f pod-iscsi.yaml

cat > pod-nas.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: csi-nas-test-pod
  namespace: test
  labels:
    app: csi-nas-test
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: test-container
      image: harbor.ocp.andy.com/dell/csi-powerstore:v2.17.0
      command: ["/bin/sh", "-c", "sleep infinity"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        runAsNonRoot: true
        runAsUser: 1000670000
      volumeMounts:
        - name: test-nas-pvc
          mountPath: /data
  volumes:
    - name: nas-test-volume
      persistentVolumeClaim:
        claimName: test-nas-pvc
EOF

oc create -f pod-nas.yaml
```
