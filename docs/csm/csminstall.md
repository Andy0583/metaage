**登入OCP**
```
oc login api.ocp.andy.com:6443 -u kubeadmin -p MAvMD-mjF22-T6rVY-euWVe

export KUBECONFIG=/home/ocp-offline/sno-install/auth/kubeconfig
```

**上傳Dell CSM Image**
```
podman login harbor.ocp.andy.com -u admin -p Harbor12345

tar xvfz dell-csm-operator-bundle.tar.gz

cd dell-csm-operator-bundle

chmod 777 * -R

bash scripts/csm-offline-bundle.sh -p -r harbor.ocp.andy.com/dell/
```

**上傳snapshot-controller Image**
```
cd

podman load -i snapshot-controller_v8.6.0.tar

podman tag registry.k8s.io/sig-storage/snapshot-controller:v8.6.0 harbor.ocp.andy.com/dell/snapshot-controller:v8.6.0

podman push harbor.ocp.andy.com/dell/snapshot-controller:v8.6.0 
```

**snapshot-controller安裝**
```
tar zxvf external-snapshotter.tar.gz

sed -i \
  's|registry.k8s.io/sig-storage/snapshot-controller:v8.6.0|harbor.ocp.andy.com/dell/snapshot-controller:v8.6.0|' \
  external-snapshotter/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml

cd external-snapshotter

oc -n kube-system kustomize deploy/kubernetes/snapshot-controller | oc create -f -

oc get deploy -n kube-system
```

**建立secret**
```
cat << EOF > secret.yaml
arrays:
  - endpoint: "https://172.22.33.24/api/rest"
    globalID: "PS3ca9ddfd760c"
    username: "admin"
    password: "**********"
    nasName: "andy-nas24"
    skipCertificateValidation: true
    isDefault: true
    blockProtocol: "ISCSI"
EOF

oc create ns powerstore

oc create secret generic powerstore-config -n powerstore --from-file=config=secret.yaml
```

**CSM Operator安裝**
```
cd ~/dell-csm-operator-bundle/scripts/

./install.sh

oc get pod -n dell-csm-operator
```

**設定Image Tag Mirror Set**
```
cat > dell-csi-tag-mirror.yaml << 'EOF'
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: dell-csi-tag-mirror
spec:
  imageTagMirrors:
    - source: registry.k8s.io/sig-storage
      mirrors:
        - harbor.ocp.andy.com/dell
    - source: quay.io/dell/container-storage-modules
      mirrors:
        - harbor.ocp.andy.com/dell
    - source: quay.io/jetstack
      mirrors:
        - harbor.ocp.andy.com/dell
    - source: quay.io/nginx
      mirrors:
        - harbor.ocp.andy.com/dell
    - source: ghcr.io/open-telemetry/opentelemetry-collector-releases
      mirrors:
        - harbor.ocp.andy.com/dell
EOF

oc apply -f dell-csi-tag-mirror.yaml

oc get mcp master -w

oc debug node/sno -- chroot /host systemctl restart crio 
```

**設定安裝yaml及啟用**
sed -i \
  -e 's/replicas: 2/replicas: 1/' \
  -e 's/value: ""/value: "192.168.131.0\/24"/' \
  ~/dell-csm-operator-bundle/samples/v2.17.0/storage_csm_powerstore_v2170.yaml

oc create -f ~/dell-csm-operator-bundle/samples/v2.17.0/storage_csm_powerstore_v2170.yaml

oc get pod -n powerstore
```


