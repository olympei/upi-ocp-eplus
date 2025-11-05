# OpenShift 4.18.23 UPI (User Provisioned Infrastructure) Installation Guide
## Bare Metal Disconnected Installation

---

## Overview

**Installation Method**: UPI (User Provisioned Infrastructure)  
**Platform**: Bare Metal  
**OpenShift Version**: 4.18.23  
**Architecture**: x86_64  
**Network**: Disconnected/Air-gapped  
**Internal Registry**: ndcu01-registry01.mnsoxen.vzbi.com:8443

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Infrastructure Services (Pre-existing)                         │
│  - DNS Server: 206.112.201.5                                   │
│  - NTP Server: 146.170.69.199                                  │
│  - DHCP/PXE Server (for bootstrap)                             │
│  - Load Balancer (HAProxy): 192.168.174.199, 192.168.174.200  │
│  - Registry: ndcu01-registry01.mnsoxen.vzbi.com:8443          │
│  - HTTP Server (for ignition files)                            │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  OpenShift Cluster Nodes                                        │
│                                                                 │
│  Bootstrap Node (temporary):                                   │
│    - 192.168.174.210                                           │
│                                                                 │
│  Control Plane Nodes:                                          │
│    - ndcu01-mstr001: 192.168.174.201                          │
│    - ndcu01-mstr002: 192.168.174.202                          │
│    - ndcu01-mstr003: 192.168.174.203                          │
│                                                                 │
│  Worker Nodes:                                                 │
│    - ndcu01-wrkr001: 192.168.174.204                          │
│    - ndcu01-wrkr002: 192.168.174.205                          │
│    - ndcu01-wrkr003: 192.168.174.206                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Infrastructure Requirements

1. **DNS Records** (must be configured before installation)
2. **Load Balancer** (HAProxy or similar)
3. **HTTP Server** (for hosting ignition files)
4. **DHCP/PXE Server** (optional, for network boot)
5. **Mirror Registry** (with OCP 4.18.23 images)

### Node Requirements

| Node Type | Count | vCPU | RAM | Storage |
|-----------|-------|------|-----|---------|
| Bootstrap | 1 | 4 | 16 GB | 120 GB |
| Master | 3 | 4 | 16 GB | 120 GB |
| Worker | 3+ | 2 | 8 GB | 120 GB |

---

## Phase 1: DNS Configuration

### Step 1.1: Required DNS Records

```bash
# API endpoint (load balanced to control plane nodes)
api.ndctc01-ocpclstr01.mnsoxen.vzbi.com        A    192.168.174.199

# API internal endpoint
api-int.ndctc01-ocpclstr01.mnsoxen.vzbi.com    A    192.168.174.199

# Ingress/Apps wildcard (load balanced to worker nodes)
*.apps.ndctc01-ocpclstr01.mnsoxen.vzbi.com     A    192.168.174.200

# etcd records (one per control plane node)
etcd-0.ndctc01-ocpclstr01.mnsoxen.vzbi.com     A    192.168.174.201
etcd-1.ndctc01-ocpclstr01.mnsoxen.vzbi.com     A    192.168.174.202
etcd-2.ndctc01-ocpclstr01.mnsoxen.vzbi.com     A    192.168.174.203

# SRV record for etcd
_etcd-server-ssl._tcp.ndctc01-ocpclstr01.mnsoxen.vzbi.com  SRV  0 10 2380 etcd-0.ndctc01-ocpclstr01.mnsoxen.vzbi.com
_etcd-server-ssl._tcp.ndctc01-ocpclstr01.mnsoxen.vzbi.com  SRV  0 10 2380 etcd-1.ndctc01-ocpclstr01.mnsoxen.vzbi.com
_etcd-server-ssl._tcp.ndctc01-ocpclstr01.mnsoxen.vzbi.com  SRV  0 10 2380 etcd-2.ndctc01-ocpclstr01.mnsoxen.vzbi.com

# Bootstrap node (temporary)
bootstrap.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A    192.168.174.210

# Control plane nodes
ndcu01-mstr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.201
ndcu01-mstr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.202
ndcu01-mstr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.203

# Worker nodes
ndcu01-wrkr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.204
ndcu01-wrkr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.205
ndcu01-wrkr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com  A  192.168.174.206

# Reverse DNS (PTR records)
201.174.168.192.in-addr.arpa  PTR  ndcu01-mstr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com
202.174.168.192.in-addr.arpa  PTR  ndcu01-mstr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com
203.174.168.192.in-addr.arpa  PTR  ndcu01-mstr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com
204.174.168.192.in-addr.arpa  PTR  ndcu01-wrkr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com
205.174.168.192.in-addr.arpa  PTR  ndcu01-wrkr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com
206.174.168.192.in-addr.arpa  PTR  ndcu01-wrkr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com
210.174.168.192.in-addr.arpa  PTR  bootstrap.ndctc01-ocpclstr01.mnsoxen.vzbi.com
```

### Step 1.2: Verify DNS Configuration

```bash
# Test API endpoint
nslookup api.ndctc01-ocpclstr01.mnsoxen.vzbi.com

# Test wildcard apps
nslookup test.apps.ndctc01-ocpclstr01.mnsoxen.vzbi.com

# Test etcd SRV records
dig _etcd-server-ssl._tcp.ndctc01-ocpclstr01.mnsoxen.vzbi.com SRV

# Test reverse DNS
nslookup 192.168.174.201
```

---

## Phase 2: Load Balancer Configuration (HAProxy)

### Step 2.1: Install HAProxy

```bash
# On load balancer host
sudo yum install -y haproxy
```

### Step 2.2: Configure HAProxy

```bash
sudo cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# API Frontend (6443)
frontend api-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend api-backend

# API Backend (Bootstrap + Masters)
backend api-backend
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server bootstrap 192.168.174.210:6443 check
    server master1 192.168.174.201:6443 check
    server master2 192.168.174.202:6443 check
    server master3 192.168.174.203:6443 check

# Machine Config Server Frontend (22623)
frontend machine-config-frontend
    bind *:22623
    mode tcp
    option tcplog
    default_backend machine-config-backend

# Machine Config Backend (Bootstrap + Masters)
backend machine-config-backend
    mode tcp
    balance roundrobin
    server bootstrap 192.168.174.210:22623 check
    server master1 192.168.174.201:22623 check
    server master2 192.168.174.202:22623 check
    server master3 192.168.174.203:22623 check

# Ingress HTTP Frontend (80)
frontend ingress-http-frontend
    bind *:80
    mode tcp
    option tcplog
    default_backend ingress-http-backend

# Ingress HTTP Backend (Workers)
backend ingress-http-backend
    mode tcp
    balance roundrobin
    server worker1 192.168.174.204:80 check
    server worker2 192.168.174.205:80 check
    server worker3 192.168.174.206:80 check

# Ingress HTTPS Frontend (443)
frontend ingress-https-frontend
    bind *:443
    mode tcp
    option tcplog
    default_backend ingress-https-backend

# Ingress HTTPS Backend (Workers)
backend ingress-https-backend
    mode tcp
    balance roundrobin
    option ssl-hello-chk
    server worker1 192.168.174.204:443 check
    server worker2 192.168.174.205:443 check
    server worker3 192.168.174.206:443 check
EOF
```

### Step 2.3: Start HAProxy

```bash
# Enable and start
sudo systemctl enable haproxy
sudo systemctl start haproxy

# Verify
sudo systemctl status haproxy

# Check ports
sudo ss -tlnp | grep haproxy
```

---

## Phase 3: HTTP Server for Ignition Files

### Step 3.1: Install HTTP Server

```bash
# Install Apache or Nginx
sudo yum install -y httpd

# Create directory for ignition files
sudo mkdir -p /var/www/html/ignition
sudo chmod 755 /var/www/html/ignition
```

### Step 3.2: Configure HTTP Server

```bash
# Start and enable
sudo systemctl enable httpd
sudo systemctl start httpd

# Open firewall
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload

# Test
curl http://localhost/
```

---

## Phase 4: Create install-config.yaml

### Step 4.1: Create Installation Directory

```bash
mkdir ~/upi-install
cd ~/upi-install
```

### Step 4.2: Create install-config.yaml

```yaml
apiVersion: v1
baseDomain: mnsoxen.vzbi.com
metadata:
  name: ndctc01-ocpclstr01

compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0  # Set to 0 for UPI - we'll add workers manually

controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3

networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  machineNetwork:
  - cidr: 192.168.174.0/24
  networkType: OVNKubernetes

platform:
  none: {}  # UPI uses 'none' platform

pullSecret: '{"auths":{"ndcu01-registry01.mnsoxen.vzbi.com:8443":{"auth":"YWRtaW46VmVyaXpvbk9jcDEyMyE="},"cloud.openshift.com":{"auth":"..."},"quay.io":{"auth":"..."},"registry.redhat.io":{"auth":"..."}}}'

sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ0C5Kmf6Se+rqp8rD66WZpANRlBLqYGo2ENxxP09X4h stephen.damson@verizon.com'

additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  [Your registry certificate]
  -----END CERTIFICATE-----

imageContentSources:
- mirrors:
  - ndcu01-registry01.mnsoxen.vzbi.com:8443/ocp4/openshift/release
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ndcu01-registry01.mnsoxen.vzbi.com:8443/ocp4/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

---

## Phase 5: Generate Ignition Files

### Step 5.1: Backup install-config.yaml

```bash
# The installer consumes this file, so back it up
cp install-config.yaml install-config.yaml.backup
```

### Step 5.2: Generate Manifests

```bash
# Generate Kubernetes manifests
openshift-install create manifests --dir ~/upi-install

# This creates:
# - manifests/
# - openshift/
```

### Step 5.3: Modify Manifests (Optional)

```bash
# Prevent masters from being schedulable (optional)
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' manifests/cluster-scheduler-02-config.yml

# Remove machine and machineset manifests (UPI doesn't use them)
rm -f openshift/99_openshift-cluster-api_master-machines-*.yaml
rm -f openshift/99_openshift-cluster-api_worker-machineset-*.yaml
```

### Step 5.4: Generate Ignition Files

```bash
# Generate ignition configs
openshift-install create ignition-configs --dir ~/upi-install

# This creates:
# - bootstrap.ign
# - master.ign
# - worker.ign
# - metadata.json
# - auth/kubeconfig
# - auth/kubeadmin-password
```

### Step 5.5: Copy Ignition Files to HTTP Server

```bash
# Copy to web server
sudo cp ~/upi-install/*.ign /var/www/html/ignition/
sudo chmod 644 /var/www/html/ignition/*.ign

# Verify accessibility
curl http://192.168.174.201/ignition/bootstrap.ign
curl http://192.168.174.201/ignition/master.ign
curl http://192.168.174.201/ignition/worker.ign
```

---

## Phase 6: Prepare RHCOS Boot Media

### Step 6.1: Download RHCOS Images

```bash
# Download RHCOS ISO and kernel/initramfs for PXE
# Version must match OCP 4.18.23

# For ISO boot
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-live.x86_64.iso

# For PXE boot
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-live-kernel-x86_64
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-live-initramfs.x86_64.img
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.18/latest/rhcos-live-rootfs.x86_64.img
```

### Step 6.2: Create Bootable Media

**Option A: ISO Boot (Easier)**

```bash
# Burn ISO to USB or mount to virtual media
# Boot parameters will be added at boot time
```

**Option B: PXE Boot (Automated)**

```bash
# Copy files to PXE server
sudo cp rhcos-live-kernel-x86_64 /var/lib/tftpboot/rhcos/
sudo cp rhcos-live-initramfs.x86_64.img /var/lib/tftpboot/rhcos/
sudo cp rhcos-live-rootfs.x86_64.img /var/www/html/rhcos/
```

---

## Phase 7: Boot Bootstrap Node

### Step 7.1: Boot Bootstrap with Ignition

**Boot Parameters:**

```bash
# For ISO boot, at boot prompt add:
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/bootstrap.ign \
coreos.inst.insecure=yes \
ip=192.168.174.210::192.168.174.1:255.255.255.0:bootstrap.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

**For PXE boot, configure in PXE menu:**

```
LABEL bootstrap
  MENU LABEL Bootstrap Node
  KERNEL rhcos/rhcos-live-kernel-x86_64
  APPEND initrd=rhcos/rhcos-live-initramfs.x86_64.img coreos.live.rootfs_url=http://192.168.174.201/rhcos/rhcos-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.174.201/ignition/bootstrap.ign ip=192.168.174.210::192.168.174.1:255.255.255.0:bootstrap.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none nameserver=206.112.201.5
```

### Step 7.2: Monitor Bootstrap

```bash
# SSH to bootstrap node
ssh core@192.168.174.210

# Watch bootstrap progress
journalctl -b -f -u release-image.service -u bootkube.service
```

---

## Phase 8: Boot Control Plane Nodes

### Step 8.1: Boot Master Nodes

Boot each master node with these parameters:

**Master 1 (192.168.174.201):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/master.ign \
coreos.inst.insecure=yes \
ip=192.168.174.201::192.168.174.1:255.255.255.0:ndcu01-mstr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

**Master 2 (192.168.174.202):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/master.ign \
coreos.inst.insecure=yes \
ip=192.168.174.202::192.168.174.1:255.255.255.0:ndcu01-mstr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

**Master 3 (192.168.174.203):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/master.ign \
coreos.inst.insecure=yes \
ip=192.168.174.203::192.168.174.1:255.255.255.0:ndcu01-mstr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

### Step 8.2: Wait for Installation

```bash
# From installation host, monitor bootstrap completion
openshift-install wait-for bootstrap-complete --dir ~/upi-install --log-level=debug

# Expected output after ~30 minutes:
# INFO Waiting up to 20m0s for the Kubernetes API at https://api.ndctc01-ocpclstr01.mnsoxen.vzbi.com:6443...
# INFO API v1.31.0 up
# INFO Waiting up to 30m0s for bootstrapping to complete...
# INFO It is now safe to remove the bootstrap resources
```

---

## Phase 9: Remove Bootstrap Node

### Step 9.1: Remove Bootstrap from Load Balancer

```bash
# Edit HAProxy config
sudo vi /etc/haproxy/haproxy.cfg

# Comment out or remove bootstrap lines:
# backend api-backend
#     server bootstrap 192.168.174.210:6443 check  # REMOVE THIS
# backend machine-config-backend
#     server bootstrap 192.168.174.210:22623 check  # REMOVE THIS

# Reload HAProxy
sudo systemctl reload haproxy
```

### Step 9.2: Power Off Bootstrap Node

```bash
# Bootstrap node is no longer needed
# Power it off or delete the VM
```

---

## Phase 10: Boot Worker Nodes

### Step 10.1: Boot Worker Nodes

Boot each worker with these parameters:

**Worker 1 (192.168.174.204):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/worker.ign \
coreos.inst.insecure=yes \
ip=192.168.174.204::192.168.174.1:255.255.255.0:ndcu01-wrkr001.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

**Worker 2 (192.168.174.205):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/worker.ign \
coreos.inst.insecure=yes \
ip=192.168.174.205::192.168.174.1:255.255.255.0:ndcu01-wrkr002.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

**Worker 3 (192.168.174.206):**
```bash
coreos.inst.install_dev=/dev/sda \
coreos.inst.ignition_url=http://192.168.174.201/ignition/worker.ign \
coreos.inst.insecure=yes \
ip=192.168.174.206::192.168.174.1:255.255.255.0:ndcu01-wrkr003.ndctc01-ocpclstr01.mnsoxen.vzbi.com:ens5f0:none \
nameserver=206.112.201.5
```

### Step 10.2: Approve Worker CSRs

```bash
# Export kubeconfig
export KUBECONFIG=~/upi-install/auth/kubeconfig

# Watch for pending CSRs
watch -n5 oc get csr

# Approve all pending CSRs
oc get csr -o name | xargs oc adm certificate approve

# Workers will appear twice (one for kubelet, one for serving cert)
# Approve both rounds
```

---

## Phase 11: Complete Installation

### Step 11.1: Wait for Installation Complete

```bash
# Monitor installation completion
openshift-install wait-for install-complete --dir ~/upi-install --log-level=debug

# Expected output after ~40 minutes:
# INFO Waiting up to 40m0s for the cluster at https://api.ndctc01-ocpclstr01.mnsoxen.vzbi.com:6443 to initialize...
# INFO Waiting up to 10m0s for the openshift-console route to be created...
# INFO Install complete!
# INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/upi-install/auth/kubeconfig'
# INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ndctc01-ocpclstr01.mnsoxen.vzbi.com
# INFO Login to the console with user: "kubeadmin", and password: "xxxxx-xxxxx-xxxxx-xxxxx"
```

### Step 11.2: Verify Cluster

```bash
# Set kubeconfig
export KUBECONFIG=~/upi-install/auth/kubeconfig

# Check nodes
oc get nodes

# Check cluster operators
oc get co

# Check cluster version
oc get clusterversion

# All operators should be Available=True, Progressing=False, Degraded=False
```

---

## Phase 12: Post-Installation Tasks

### Step 12.1: Configure Image Registry Storage

```bash
# For disconnected, configure emptyDir (not for production)
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'

# Set managementState to Managed
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'

# For production, configure PV storage
```

### Step 12.2: Configure Registry for Disconnected

```bash
# The imageContentSources from install-config.yaml are already applied
# Verify
oc get imagecontentsourcepolicy

# Check nodes have the mirror configuration
oc debug node/ndcu01-mstr001 -- chroot /host cat /etc/containers/registries.conf
```

### Step 12.3: Save Credentials

```bash
# Save kubeadmin password
cat ~/upi-install/auth/kubeadmin-password

# Save kubeconfig
cp ~/upi-install/auth/kubeconfig ~/.kube/config
```

---

## Troubleshooting

### Bootstrap Not Completing

```bash
# SSH to bootstrap
ssh core@192.168.174.210

# Check logs
journalctl -b -f -u bootkube.service

# Check if API is accessible
curl -k https://api.ndctc01-ocpclstr01.mnsoxen.vzbi.com:6443/version
```

### Masters Not Joining

```bash
# Check master logs
ssh core@192.168.174.201
journalctl -b -f -u kubelet.service

# Check if ignition was applied
sudo cat /etc/hostname
```

### Workers Not Joining

```bash
# Check pending CSRs
oc get csr | grep Pending

# Approve manually
oc adm certificate approve <csr-name>

# Or approve all
oc get csr -o name | xargs oc adm certificate approve
```

### Image Pull Failures

```bash
# Check imagecontentsourcepolicy
oc get imagecontentsourcepolicy -o yaml

# Check node registry config
oc debug node/ndcu01-mstr001 -- chroot /host cat /etc/containers/registries.conf

# Verify registry is accessible
curl -k https://ndcu01-registry01.mnsoxen.vzbi.com:8443/v2/
```

---

## Installation Timeline

| Phase | Duration | Description |
|-------|----------|-------------|
| Bootstrap boot | 5-10 min | Bootstrap node boots and starts services |
| Bootstrap complete | 20-30 min | API becomes available, etcd forms |
| Masters boot | 10-15 min | Control plane nodes join cluster |
| Workers boot | 10-15 min | Worker nodes join cluster |
| Operators deploy | 20-30 min | All cluster operators become ready |
| **Total** | **60-90 min** | Complete installation time |

---

## Summary

**UPI Installation Complete!**

Your cluster is now ready at:
- **API**: https://api.ndctc01-ocpclstr01.mnsoxen.vzbi.com:6443
- **Console**: https://console-openshift-console.apps.ndctc01-ocpclstr01.mnsoxen.vzbi.com
- **Username**: kubeadmin
- **Password**: (saved in ~/upi-install/auth/kubeadmin-password)

**Next Steps:**
1. Configure persistent storage for registry
2. Set up authentication (LDAP, OAuth, etc.)
3. Configure monitoring and logging
4. Deploy applications
