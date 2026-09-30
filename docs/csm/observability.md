**安裝OTel Collector**
```
cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: quay-mirror
spec:
  imageTagMirrors:
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
```
```
oc get mcp
```
```
podman load -i otel-collector.tar.gz
podman tag  ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector:0.143.1 harbor.ocp.andy.com/dell/opentelemetry-collector:0.143.1 
podman push harbor.ocp.andy.com/dell/opentelemetry-collector:0.143.1
```

**設定OTel Collector**
```
oc get csm powerstore -n powerstore -o json > /tmp/csm-current.json
```
```
python3 << 'PYEOF'
import json

with open('/tmp/csm-current.json') as f:
    csm = json.load(f)

for m in csm['spec']['modules']:
    if m['name'] == 'observability':
        m['enabled'] = True
        for c in m['components']:
            if c['name'] in ('otel-collector', 'cert-manager', 'metrics-powerstore'):
                c['enabled'] = True

with open('/tmp/csm-updated.json', 'w') as f:
    json.dump(csm, f)
PYEOF
```
```
oc apply -f /tmp/csm-updated.json
```
```
oc get pods -n powerstore
```

**建立 User Workload Monitoring**
```
cat << EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```
```
oc get pods -n openshift-user-workload-monitoring
```

**建立 Service Monitor**
```
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector-monitor
  namespace: powerstore
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: otel-collector
  endpoints:
  - port: exporter-https
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
    path: /metrics
    interval: 30s
EOF
```
```
oc get svc -n powerstore -l app.kubernetes.io/name=otel-collector
```

**相關查詢**
```
oc whoami --show-console
cat /home/ocp-offline/sno-install/auth/kubeadmin-password
```
