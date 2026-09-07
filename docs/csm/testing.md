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
