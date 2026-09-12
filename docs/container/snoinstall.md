### **<font color="red">Bastion 前置作業</font>**
**指定ISO為repo**
```
mkdir /var/repo

mount -o loop rhel-baseos-9.1-x86_64-dvd.iso /var/repo/

cat > /etc/yum.repos.d/rhel9-local.repo << EOF
[Local-BaseOS]
name=Red Hat Enterprise Linux 9 - BaseOS
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///var/repo//BaseOS/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
[Local-AppStream]
name=Red Hat Enterprise Linux 9 - AppStream
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///var/repo//AppStream/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

yum clean all

subscription-manager clean
```

**關閉防火牆與安全性**
```
systemctl stop firewalld

systemctl disable firewalld

setenforce 0

sed -i 's/enforcing/disabled/' /etc/selinux/config

sed -i 's/enabled=1/enabled=0/' /etc/yum/pluginconf.d/subscription-manager.conf
```

### **<font color="red">Infra 安裝</font>**
**安裝相關tool**
```
mkdir -p /home/ocp-offline

tar xvf ocp-offline-bundle.tar -C /home/ocp-offline

cd /home/ocp-offline/tools

tar xzf openshift-install-linux.tar.gz

tar xzf openshift-client-linux.tar.gz

tar xzf oc-mirror.tar.gz

chmod +x oc-mirror openshift-install oc kubectl

mv oc-mirror openshift-install oc kubectl /usr/local/bin/
```

**安裝NTP Server**
```
rm /etc/chrony.conf -f

cat > /etc/chrony.conf << 'EOF'
driftfile /var/lib/chrony/drift
allow 172.22.46.0/24
local stratum 10
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

timedatectl set-ntp no

timedatectl set-time "2026-09-12 22:50:00"

systemctl start chronyd

systemctl enable --now chronyd
```

**安裝DNS Server**
```
cd

mkdir -p /opt/bind-offline

tar xzvf bind-offline.tar.gz -C /opt/bind-offline

cd /opt/bind-offline

dnf localinstall *.rpm -y
```
```
vi /etc/named.conf
        listen-on port 53 { any; };
#       listen-on-v6 port 53 { ::1; };
        allow-query     { any; };
        dnssec-enable no;
        dnssec-validation no;
zone "ocp.andy.com" IN {
        type master;
        file "named.ocp.andy.com";
};
zone "46.22.172.in-addr.arpa" IN {
        type master;
        file "rev.46.22.172";
};
```
```
vi /var/named/named.ocp.andy.com

$TTL 1D
@       IN SOA  @ bastion.ocp.andy.com. (
                                        2020040819      ; serial
                                        3H      ; refresh
                                        15M     ; retry
                                        1W      ; expire
                                        1D )    ; minimum
@       IN NS   bastion.ocp.andy.com.
@       IN A    172.22.46.200

bastion                 IN      A       172.22.46.200
harbor                  IN      A       172.22.46.209
sno                     IN      A       172.22.46.201
api                     IN      A       172.22.46.201
api-int                 IN      A       172.22.46.201
*.apps                  IN      A       172.22.46.201
```
```
vi /var/named/rev.46.22.172

$TTL 1D
@       IN SOA  @ bastion.ocp.andy.com. (
                                        2020040819      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
@       IN NS   bastion.ocp.andy.com.

46.22.172.in-addr.arpa      IN      PTR     bastion.ocp.andy.com
200   IN      PTR     bastion.ocp.andy.com.
209   IN      PTR     harbor.ocp.andy.com.
201   IN      PTR     sno.ocp.andy.com.
```
```
chgrp named /var/named/named.ocp.andy.com

chmod 640 /var/named/named.ocp.andy.com

chgrp named /var/named/rev.46.22.172

chmod 640 /var/named/rev.46.22.172

systemctl enable named

systemctl start named
```

**安裝nmstate-rpms**
```
cd

cd nmstate-rpms/

rpm -Uvh *.rpm
```

### **<font color="red">OCP設定</font>**
**合併 Harbor 登入資訊**
```
python3 << 'PYEOF'
import json, base64
with open('/home/ocp-offline/pull-secret.json') as f:
    d = json.load(f)
user = 'admin'
password = 'Harbor12345'
token = base64.b64encode(f'{user}:{password}'.encode()).decode()
d['auths']['harbor.ocp.andy.com'] = {'auth': token}

with open('/home/ocp-offline/pull-secret-merged.json', 'w') as f:
    json.dump(d, f)
PYEOF
```

**Image Mirror**
```
scp root@harbor.ocp.andy.com:/data/harbor/certs/ca.crt ca.crt

mkdir -p /etc/containers/certs.d/harbor.ocp.andy.com

cp ca.crt /etc/containers/certs.d/harbor.ocp.andy.com/ca.crt

cp ca.crt  /home/ocp-offline/harbor-ca.crt

cd /home/ocp-offline

oc mirror -c mirror/imageset-config.yaml \
  --from file:///home/ocp-offline/mirror/output \
  docker://harbor.ocp.andy.com/ocp4 \
  --authfile /home/ocp-offline/pull-secret-merged.json \
  --cache-dir /home/oc-mirror-cache \
  --v2
```

**合併 pull-secret**
```
cd

mkdir -p /home/ocp-offline/sno-install

mv pull-secret.txt /home/ocp-offline/pull-secret.json -f

python3 << 'PYEOF'
import json, base64
with open('/home/ocp-offline/pull-secret.json') as f:
    d = json.load(f)
user = 'admin'
password = 'Harbor12345'
token = base64.b64encode(f'{user}:{password}'.encode()).decode()
d['auths']['harbor.ocp.andy.com'] = {'auth': token}
with open('/home/ocp-offline/sno-install/pull-secret-merged.json', 'w') as f:
    json.dump(d, f)
PYEOF
```

**產生 install-config.yaml**
```
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

python3 << 'PYEOF'
with open('/root/.ssh/id_rsa.pub') as f:
    ssh_key = f.read().strip()
with open('/home/ocp-offline/sno-install/pull-secret-merged.json') as f:
    pull_secret = f.read().strip()
with open('/etc/containers/certs.d/harbor.ocp.andy.com/ca.crt') as f:
    ca_cert = f.read().strip()
ca_indented = '\n'.join('  ' + line for line in ca_cert.split('\n'))
config = f"""apiVersion: v1
baseDomain: andy.com
metadata:
  name: ocp
compute:
  - name: worker
    replicas: 0
controlPlane:
  name: master
  replicas: 1
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 10.10.0.0/16
  machineNetwork:
    - cidr: 172.22.46.0/24
platform:
  none: {{}}
pullSecret: '{pull_secret}'
sshKey: '{ssh_key}'
additionalTrustBundlePolicy: Always
additionalTrustBundle: |
{ca_indented}
imageContentSources:
  - source: quay.io/openshift-release-dev/ocp-release
    mirrors:
      - harbor.ocp.andy.com/ocp4/openshift/release-images
  - source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
    mirrors:
      - harbor.ocp.andy.com/ocp4/openshift/release
  - source: registry.redhat.io/openshift-logging
    mirrors:
      - harbor.ocp.andy.com/ocp4/openshift-logging
  - source: registry.redhat.io/openshift4
    mirrors:
      - harbor.ocp.andy.com/ocp4/openshift4
"""
with open('/home/ocp-offline/sno-install/install-config.yaml', 'w') as f:
    f.write(config)
PYEOF
```

**產生 agent-config.yaml**
```
cat > /home/ocp-offline/sno-install/agent-config.yaml << 'EOF'
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: sno
rendezvousIP: 172.22.46.201
additionalNTPSources:
  - 172.22.46.200
hosts:
  - hostname: sno
    role: master
    interfaces:
      - name: ens33
        macAddress: "00:50:56:a8:14:88"
    networkConfig:
      interfaces:
        - name: ens33
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 172.22.46.201
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 172.22.46.200
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 172.22.46.254
            next-hop-interface: ens33
EOF
```

**停用簽章驗證**
```
mkdir -p /home/ocp-offline/sno-install/openshift

cat <<'EOF' > /tmp/policy.json
{
  "default": [{"type": "insecureAcceptAnything"}],
  "transports": {
    "docker-daemon": {"": [{"type": "insecureAcceptAnything"}]}
  }
}
EOF

POLICY_B64=$(base64 -w0 /tmp/policy.json)

for ROLE in master worker; do
cat > /home/ocp-offline/sno-install/openshift/99z-${ROLE}-zzz-disable-signature-verification.yaml << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: ${ROLE}
  name: 99z-${ROLE}-zzz-disable-signature-verification
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/containers/policy.json
          mode: 0644
          overwrite: true
          contents:
            source: data:text/plain;charset=utf-8;base64,${POLICY_B64}
EOF
done
```

**產生agent.x86_64.iso**
```
openshift-install --dir=/home/ocp-offline/sno-install agent create image --log-level=info
```

### **<font color="red">建立OCP</font>**
**設定OCP VM**
- disk.EnableUUID = TRUE
```
nohup openshift-install --dir=/home/ocp-offline/sno-install agent wait-for bootstrap-complete --log-level=info > /home/ocp-offline/sno-bootstrap.log 2>&1 &

tail -f /home/ocp-offline/sno-bootstrap.log

# 於cluster bootstrap is complete之後執行
nohup openshift-install --dir=/home/ocp-offline/sno-install agent wait-for install-complete --log-level=info > /home/ocp-offline/sno-install.log 2>&1 &

tail -f /home/ocp-offline/sno-install.log
```

**測試登入OCP**
```
oc login api.ocp.andy.com:6443 -u kubeadmin -p MAvMD-mjF22-T6rVY-euWVe

export KUBECONFIG=/home/ocp-offline/sno-install/auth/kubeconfig
```

**建立網路**
```
oc debug node/sno -- chroot /host nmcli con add \
  type ethernet \
  con-name iscsi-storage \
  ifname ens34 \
  ipv4.method manual \
  ipv4.addresses 192.168.130.201/24 \
  802-3-ethernet.mtu 9000 \
  connection.autoconnect yes

oc debug node/sno -- chroot /host nmcli con up iscsi-storage

oc debug node/sno -- chroot /host ping -c 4 -I ens34 192.168.130.250

oc debug node/sno -- chroot /host nmcli con add \
  type ethernet \
  con-name nfs-storage \
  ifname ens35 \
  ipv4.method manual \
  ipv4.addresses 192.168.131.201/24 \
  802-3-ethernet.mtu 9000 \
  connection.autoconnect yes

oc debug node/sno -- chroot /host nmcli con up nfs-storage

oc debug node/sno -- chroot /host ping -c 4 -I ens35 192.168.131.205
```
