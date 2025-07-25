---
title: "Parte 4: Setup Pratico - Da Zero a Cluster Funzionante"
date: 2025-07-24T10:30:00+01:00
description: Guida completa al deployment e gestione di cluster Kubernetes utilizzando Cluster API (CAPI) per l'automazione dell'infrastruttura
menu:
  sidebar:
    name: Setup Pratico
    identifier: CAPI-4
    weight: 25
    parent: CAPI
tags: ["Kubernetes", "CAPI", "Cluster API", "Infrastructure as Code", "DevOps", "Automazione"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

# Parte 4: Setup Pratico - Da Zero a Cluster Funzionante

*Quarto articolo della serie "Deploy Kubernetes con Cluster API: Gestione Automatizzata dei Cluster"*

---

Nelle parti precedenti abbiamo esplorato i fondamenti teorici di Cluster API, l'architettura dei componenti e l'integrazione con Talos Linux. È ora il momento di mettere in pratica questi concetti attraverso un'implementazione completa end-to-end.

Questa parte guiderà attraverso ogni step del processo: dalla configurazione dell'infrastruttura Proxmox al deployment del primo cluster workload funzionante, utilizzando il Python generator per automatizzare la generazione delle configurazioni parametriche.

---

## Monitoring e Validazione

### Cluster Health Monitoring

Una volta deployato il cluster, è fondamentale implementare monitoring continuo per assicurare stabilità operativa.

#### Resource Monitoring

```bash
# Monitor node resources
kubectl --kubeconfig kubeconfig-homelab top nodes

# Monitor pod resource usage
kubectl --kubeconfig kubeconfig-homelab top pods -A

# Check cluster capacity
kubectl --kubeconfig kubeconfig-homelab describe nodes | grep -A 5 "Allocated resources"
```

#### System Pods Health

```bash
# Check critical system pods
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -o wide

# Verify DNS resolution
kubectl --kubeconfig kubeconfig-homelab run dns-test --image=busybox --restart=Never -- nslookup kubernetes.default

# Check logs for any errors
kubectl --kubeconfig kubeconfig-homelab logs -n kube-system -l k8s-app=kube-dns
```

### CAPI Management Monitoring

Dal management cluster, monitorare lo stato dei workload clusters:

```bash
# Check cluster status from management cluster
kubectl get clusters,machines,machinedeployments -A -o wide

# Monitor CAPI controller health
kubectl get pods -n capi-system -o wide
kubectl get pods -n capx-system -o wide
kubectl get pods -n capi-control-plane-talos-system -o wide

# Check for any reconciliation errors
kubectl get events --sort-by='.lastTimestamp' -A | grep -i error
```

### Automated Health Checks

Implementare health checks automatizzati per detection proattiva di issues:

```bash
# Create health check script
cat > cluster-health-check.sh << 'EOF'
#!/bin/bash
set -e

KUBECONFIG="kubeconfig-homelab"
CLUSTER_NAME="homelab-cluster"

echo "=== Cluster Health Check for $CLUSTER_NAME ==="
echo "Timestamp: $(date)"

# Check node status
echo "Node Status:"
kubectl --kubeconfig $KUBECONFIG get nodes -o wide

# Check system pods
echo -e "\nSystem Pods:"
kubectl --kubeconfig $KUBECONFIG get pods -n kube-system | grep -v Running && echo "❌ Some system pods not running" || echo "✅ All system pods running"

# Check cluster components
echo -e "\nCluster Components:"
kubectl --kubeconfig $KUBECONFIG get componentstatuses 2>/dev/null || echo "⚠️  ComponentStatus API deprecated in this version"

# Test DNS
echo -e "\nDNS Test:"
kubectl --kubeconfig $KUBECONFIG run dns-test-$(date +%s) --image=busybox --restart=Never --rm -i --tty -- nslookup kubernetes.default || echo "❌ DNS resolution failed"

# Check API responsiveness
echo -e "\nAPI Server Response Time:"
time kubectl --kubeconfig $KUBECONFIG get nodes >/dev/null

echo -e "\n=== Health Check Complete ==="
EOF

chmod +x cluster-health-check.sh
./cluster-health-check.sh
```

---

## Backup e Disaster Recovery

### Backup Strategy

#### Management Cluster Backup

```bash
# Backup management cluster configuration
mkdir -p backups/management-cluster/$(date +%Y-%m-%d)

# Export all CAPI resources
kubectl get clusters,machines,machinedeployments,taloscontrolplanes,proxmoxclusters,proxmoxmachines -A -o yaml > backups/management-cluster/$(date +%Y-%m-%d)/capi-resources.yaml

# Backup cluster configurations
cp homelab-cluster.yaml backups/management-cluster/$(date +%Y-%m-%d)/
cp kubeconfig-homelab backups/management-cluster/$(date +%Y-%m-%d)/

# Backup environment configuration
env | grep -E "(PROXMOX|CLUSTER)" > backups/management-cluster/$(date +%Y-%m-%d)/environment.env
```

#### Workload Cluster Backup

```bash
# Backup workload cluster resources
mkdir -p backups/workload-cluster/$(date +%Y-%m-%d)

# Export critical resources
kubectl --kubeconfig kubeconfig-homelab get all -A -o yaml > backups/workload-cluster/$(date +%Y-%m-%d)/all-resources.yaml

# Backup persistent volumes
kubectl --kubeconfig kubeconfig-homelab get pv,pvc -A -o yaml > backups/workload-cluster/$(date +%Y-%m-%d)/storage.yaml

# Backup secrets and configmaps
kubectl --kubeconfig kubeconfig-homelab get secrets,configmaps -A -o yaml > backups/workload-cluster/$(date +%Y-%m-%d)/config.yaml
```

#### Proxmox VM Snapshots

```bash
# Create automated snapshot script
cat > create-cluster-snapshots.sh << 'EOF'
#!/bin/bash
SNAPSHOT_NAME="cluster-backup-$(date +%Y%m%d-%H%M)"

# Get all cluster VMs
VMS=$(qm list | grep "homelab-cluster" | awk '{print $1}')

for VM in $VMS; do
    echo "Creating snapshot for VM $VM..."
    qm snapshot $VM $SNAPSHOT_NAME --description "Automated cluster backup"
    
    # Keep only last 7 snapshots
    qm listsnapshot $VM | grep "cluster-backup" | head -n -7 | while read snapshot; do
        snap_name=$(echo $snapshot | awk '{print $2}')
        echo "Removing old snapshot: $snap_name"
        qm delsnapshot $VM $snap_name
    done
done
EOF

chmod +x create-cluster-snapshots.sh
```

### Disaster Recovery Procedures

#### Workload Cluster Recovery

```bash
# In caso di perdita completa del workload cluster
# 1. Ricreare cluster con stessa configurazione
python cluster_generator.py --config homelab.yaml --output homelab-cluster-recovery.yaml
kubectl apply -f homelab-cluster-recovery.yaml

# 2. Attendere che il cluster sia pronto
kubectl wait --for=condition=ControlPlaneReady cluster/homelab-cluster --timeout=20m

# 3. Restore resources dal backup
kubectl --kubeconfig kubeconfig-homelab apply -f backups/workload-cluster/latest/all-resources.yaml
```

#### Management Cluster Recovery

```bash
# Ricreare management cluster
kind delete cluster --name capi-management
kind create cluster --config kind-config.yaml

# Reinitialize CAPI
source ~/.capi-env
clusterctl init --infrastructure proxmox --ipam in-cluster --control-plane talos --bootstrap talos

# Restore CAPI resources
kubectl apply -f backups/management-cluster/latest/capi-resources.yaml
```

---

## Performance Tuning e Optimization

### Resource Optimization

#### Control Plane Tuning

Per ambienti con risorse limitate, ottimizzare le configurazioni:

```yaml
# Configurazione ottimizzata per homelab
machine_template:
  memory_mib: 2048  # Minimum per control plane
  cpu:
    cores: 2        # Sufficiente per single control plane
  
# Talos optimizations
talos_config:
  kernel_args:
    - "net.ifnames=0"
    - "cgroup_enable=memory"
    - "cgroup_memory=1"
    - "systemd.unified_cgroup_hierarchy=0"  # Per compatibility
```

#### Worker Node Optimization

```yaml
# Worker configuration per workload diversity
workers:
  machine_template:
    memory_mib: 4096  # 4GB baseline per containers
    cpu:
      cores: 2        # Scale basato su workload
    disks:
      boot_volume:
        size_gb: 30   # Include spazio per container images
        
# Kubelet optimizations in Talos config
talos_config:
  configPatches:
    - op: "add"
      path: "/machine/kubelet/extraArgs"
      value:
        max-pods: "110"           # Default Kubernetes
        pods-per-core: "10"       # Limit pods per core
        kube-reserved: "cpu=100m,memory=128Mi"
        system-reserved: "cpu=100m,memory=128Mi"
```

### Network Performance

#### CNI Optimization

```bash
# Per Calico, ottimizzare per small clusters
kubectl --kubeconfig kubeconfig-homelab patch installation default --type merge -p '{"spec":{"calicoNetwork":{"mtu":1440}}}'

# Disable IP-in-IP se non necessario
kubectl --kubeconfig kubeconfig-homelab patch ippool default-ipv4-ippool --type merge -p '{"spec":{"ipipMode":"Never"}}'
```

#### Load Balancer Tuning

```yaml
# MetalLB configuration ottimizzata
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.40-192.168.0.45  # Reduced range per homelab
  autoAssign: true
  avoidBuggyIPs: true
```

### Storage Performance

#### Local Storage Optimization

```bash
# Configure local-path-provisioner per performance
kubectl --kubeconfig kubeconfig-homelab patch configmap local-path-config -n local-path-storage --type merge -p '{
  "data": {
    "config.json": "{\"nodePathMap\":[{\"node\":\"DEFAULT_PATH_FOR_NON_LISTED_NODES\",\"paths\":[\"/var/lib/rancher/local-path-provisioner\"]}],\"sharedFileSystemPath\":\"/var/lib/rancher/local-path-provisioner\"}"
  }
}'
```

---

## Security Hardening

### Cluster Security

#### Network Policies

```bash
# Implement baseline network policies
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
EOF
```

#### Pod Security Standards

```bash
# Enable Pod Security Standards
kubectl --kubeconfig kubeconfig-homelab label namespace default \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

### Talos Security Configuration

Talos fornisce security-by-default, ma alcune ottimizzazioni aggiuntive:

```yaml
# Enhanced Talos security config
talos_config:
  configPatches:
    - op: "add"
      path: "/machine/logging"
      value:
        destinations:
          - endpoint: "tcp://192.168.0.10:514"  # Syslog server
            format: "json_lines"
    
    - op: "add"
      path: "/machine/audit"
      value:
        logfile: "/var/log/audit.log"
        maxSize: 100
        maxAge: 30
        maxBackups: 3
        
    - op: "add"
      path: "/machine/sysctls"
      value:
        net.core.bpf_jit_harden: "2"
        kernel.kptr_restrict: "2"
        kernel.dmesg_restrict: "1"
```

---

## Troubleshooting Common Issues

### Deployment Failures

#### VM Creation Issues

```bash
# Debug Proxmox machine creation
kubectl describe proxmoxmachine homelab-cluster-cp-xyz

# Common issues and solutions:
# 1. Template not found
curl -k -H "Authorization: PVEAPIToken=$PROXMOX_TOKEN=$PROXMOX_SECRET" \
     "$PROXMOX_URL/nodes/K8S0/qemu/8700/config"

# 2. Insufficient resources
qm status 8700
pvesh get /nodes/K8S0/status

# 3. Network configuration
ip link show vmbr0
```

#### Bootstrap Problems

```bash
# Debug Talos bootstrap
kubectl get talosconfig homelab-cluster-cp-xyz -o yaml

# Check cloud-init logs (se accessibile via console)
# tail -f /var/log/cloud-init-output.log

# Verify Talos API connectivity
talosctl -n 192.168.0.21 version --insecure
```

#### Control Plane Issues

```bash
# Debug control plane provider
kubectl logs -n capi-control-plane-talos-system deployment/capi-control-plane-talos-controller-manager

# Check etcd status
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -l component=etcd

# API server connectivity
curl -k https://192.168.0.30:6443/healthz
```

### Runtime Issues

#### Node Not Ready

```bash
# Diagnose node issues
kubectl --kubeconfig kubeconfig-homelab describe node homelab-cluster-cp-xyz

# Common causes:
# 1. CNI not installed/working
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -l k8s-app=calico-node

# 2. Disk pressure
kubectl --kubeconfig kubeconfig-homelab get events --field-selector reason=NodeHasDiskPressure

# 3. Memory pressure  
kubectl --kubeconfig kubeconfig-homelab top nodes
```

#### Pod Scheduling Issues

```bash
# Debug scheduling problems
kubectl --kubeconfig kubeconfig-homelab describe pod problem-pod

# Check resource constraints
kubectl --kubeconfig kubeconfig-homelab describe nodes | grep -A 5 "Allocated resources"

# Verify taints and tolerations
kubectl --kubeconfig kubeconfig-homelab get nodes -o json | jq '.items[].spec.taints'
```

### Network Connectivity Problems

```bash
# Test pod-to-pod communication
kubectl --kubeconfig kubeconfig-homelab run netshoot --image=nicolaka/netshoot -it --rm

# Test DNS resolution
kubectl --kubeconfig kubeconfig-homelab run dns-debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check service endpoints
kubectl --kubeconfig kubeconfig-homelab get endpoints -A
```

---

## Next Steps e Best Practices

### Cluster Lifecycle Management

Una volta deployato il primo cluster, considerare:

#### Multi-Cluster Management

```bash
# Deploy additional clusters per different environments
python cluster_generator.py --cluster-name "staging-cluster" --control-plane-ip "192.168.0.35" --output staging.yaml
python cluster_generator.py --cluster-name "dev-cluster" --replicas 1 --workers-disabled --control-plane-ip "192.168.0.36" --output dev.yaml
```

#### GitOps Integration

```bash
# Structure repository per GitOps
mkdir -p clusters/{management,homelab,staging,dev}
mv homelab-cluster.yaml clusters/homelab/
cp kubeconfig-homelab clusters/homelab/

# Initialize Git repository
git init
git add clusters/
git commit -m "Initial cluster configurations"
```

#### Monitoring e Observability

```bash
# Install Prometheus stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl --kubeconfig kubeconfig-homelab create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kubeconfig kubeconfig-homelab
```

### Production Readiness Checklist

Prima di utilizzare in production:

- [ ] **High Availability**: Deploy control plane con 3+ replicas
- [ ] **Backup Strategy**: Automated snapshots e resource backups
- [ ] **Monitoring**: Prometheus/Grafana stack deployed
- [ ] **Security**: Network policies, Pod Security Standards, RBAC
- [ ] **Storage**: Persistent volume strategy (local-path, Longhorn, etc.)
- [ ] **Networking**: LoadBalancer, Ingress controller configured
- [ ] **Documentation**: Runbooks per operations comuni
- [ ] **Testing**: Disaster recovery procedures testate

---

L'implementazione pratica di Cluster API con Talos Linux su Proxmox dimostra come l'automazione dichiarativa possa trasformare la gestione dell'infrastruttura Kubernetes. Il Python generator fornisce flessibilità nella configurazione, mentre CAPI garantisce consistency e riproducibilità.

Il cluster deployato rappresenta una base solida per sperimentazione avanzata e eventuale utilizzo production. La combinazione di Talos Linux per immutabilità e sicurezza, Proxmox per controllo dell'infrastruttura, e CAPI per orchestrazione, crea un ecosistema potente e maintainable.

*La prossima parte esplorerà la gestione avanzata del cluster, incluso scaling dinamico, upgrade procedures, worker node management, e best practices operative per environments production.* Prerequisiti e Architettura Target

### Overview dell'Implementazione

L'implementazione pratica seguirà questa architettura:

```
┌─────────────────────┐    ┌──────────────────────┐    ┌─────────────────────┐
│  Management Cluster │    │   Proxmox VE         │    │  Workload Cluster   │
│  (Kind)             │───▶│   Infrastructure     │───▶│  (Talos Linux)      │
│  - CAPI Controllers │    │   - VM Templates     │    │  - Control Plane    │
│  - Python Generator │    │   - Networking       │    │  - Worker Nodes     │
│  - Provider Config  │    │   - Storage          │    │  - Applications     │
└─────────────────────┘    └──────────────────────┘    └─────────────────────┘
```

### Requisiti Hardware e Software

#### Proxmox VE Requirements
- **Proxmox VE 8.x** con accesso API abilitato
- **Minimum 32GB RAM** per l'host Proxmox (8GB + 4GB per VM + overhead)
- **4+ CPU cores** per performance accettabili
- **100GB+ storage** disponibile per VM (SSD raccomandato)
- **Network bridge** configurato (tipicamente `vmbr0`)

#### Management Environment
- **Linux/macOS workstation** con accesso di rete a Proxmox
- **Docker** installato per Kind cluster
- **Python 3.7+** per il generator
- **kubectl** e **clusterctl** CLI tools

#### Network Configuration
```bash
# Esempio configurazione di rete homelab
Proxmox Host: 192.168.0.10/24
Management Workstation: 192.168.0.5/24
Control Plane VIP: 192.168.0.30/24
Cluster IP Range: 192.168.0.20-192.168.0.49/24
Gateway: 192.168.0.1
DNS: 8.8.8.8, 8.8.4.4
```

---

## Configurazione Proxmox VE

### Setup API User e Permissions

Proxmox richiede un utente dedicato con permissions appropriate per l'automazione CAPI.

#### Creazione User e Token

```bash
# SSH al Proxmox host
ssh root@192.168.0.10

# Creazione utente CAPI
pveum user add capi@pve --comment "Cluster API Automation User"

# Assignment ruolo Administrator
pveum aclmod / -user capi@pve -role Administrator

# Generazione API token
pveum user token add capi@pve capi-token --privsep 0
```

**Output atteso:**
```console
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ capi@pve!capi-token                  │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ 12345678-1234-1234-1234-123456789abc │
└──────────────┴──────────────────────────────────────┘
```

#### Verifica API Access

```bash
# Test API connectivity
curl -k -H "Authorization: PVEAPIToken=capi@pve!capi-token=12345678-1234-1234-1234-123456789abc" \
     "https://192.168.0.10:8006/api2/json/version"

# Expected response
{
  "data": {
    "version": "8.2.2",
    "repoid": "06a4bc2e6",
    "release": "8.2"
  }
}
```

### Creazione Talos Template

Il template VM rappresenta l'immagine base che verrà clonata per ogni nodo del cluster.

#### Download Talos Image Ottimizzata

```bash
# Talos factory image con estensioni per Proxmox
wget https://factory.talos.dev/image/\
ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/\
v1.9.5/metal-amd64.iso \
-O /var/lib/vz/template/iso/talos-v1.9.5-proxmox.iso
```

Questa immagine include:
- **QEMU Guest Agent** per comunicazione VM-host
- **NoCloud datasource** per cloud-init integration
- **Optimized kernel** per virtualizzazione

#### Template VM Creation

```bash
# Create template VM
qm create 8700 \
  --name "talos-v1.9.5-template" \
  --description "Talos Linux v1.9.5 template for CAPI" \
  --ostype l26 \
  --memory 2048 \
  --balloon 0 \
  --cores 2 \
  --cpu cputype=host \
  --machine q35 \
  --bios ovmf \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --scsi0 local-lvm:20,format=qcow2,cache=writeback,discard=on \
  --scsihw virtio-scsi-pci \
  --boot order=scsi0 \
  --agent enabled=1,fstrim_cloned_disks=1 \
  --tablet 0 \
  --hotplug network,disk,usb \
  --tags "kubernetes,talos,template"

# Add EFI disk per UEFI boot
qm set 8700 --efidisk0 local-lvm:1,format=qcow2,efitype=4m,pre-enrolled-keys=1

# Attach Talos ISO
qm set 8700 --ide2 local:iso/talos-v1.9.5-proxmox.iso,media=cdrom

# Convert to template
qm template 8700
```

#### Template Validation

```bash
# Verify template creation
qm list | grep 8700
# Output: 8700 talos-v1.9.5-template   0    2048      0.00     20.00 template

# Check template configuration
qm config 8700 | grep -E "(name|template|agent|net0)"
```

### Network Bridge Configuration

Assicurarsi che il bridge di rete sia configurato correttamente per l'accesso dei cluster nodes.

#### Bridge Verification

```bash
# Check existing bridges
ip link show type bridge

# Verify bridge configuration
cat /etc/network/interfaces | grep -A 10 vmbr0

# Example expected output:
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.10/24
    gateway 192.168.0.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

#### Firewall Rules (Optional)

```bash
# Allow Kubernetes API traffic
iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
iptables -A FORWARD -p tcp --dport 6443 -j ACCEPT

# Allow pod-to-pod communication
iptables -A FORWARD -s 192.168.0.0/24 -d 192.168.0.0/24 -j ACCEPT

# Persist rules
iptables-save > /etc/iptables/rules.v4
```

---

## Management Cluster Setup

### Kind Cluster Creation

Il management cluster serve come control plane per orchestrare i workload clusters.

#### Kind Configuration

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: capi-management
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "node-role.kubernetes.io/management=true"
  extraPortMappings:
  # Expose CAPI webhook ports
  - containerPort: 9443
    hostPort: 9443
    protocol: TCP
```

```bash
# Create management cluster
kind create cluster --config kind-config.yaml

# Verify cluster
kubectl cluster-info --context kind-capi-management
kubectl get nodes -o wide
```

### Tools Installation

#### clusterctl Installation

```bash
# Download latest clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.10.3/clusterctl-linux-amd64 -o clusterctl

# Install
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl

# Verify installation
clusterctl version
```

#### Provider Configuration

```bash
# Create clusterctl configuration directory
mkdir -p ~/.cluster-api

# Provider configuration
cat > ~/.cluster-api/clusterctl.yaml << EOF
providers:
  - name: "talos"
    url: "https://github.com/siderolabs/cluster-api-bootstrap-provider-talos/releases/v0.6.7/bootstrap-components.yaml"
    type: "BootstrapProvider"
  - name: "talos"
    url: "https://github.com/siderolabs/cluster-api-control-plane-provider-talos/releases/v0.5.8/control-plane-components.yaml"
    type: "ControlPlaneProvider"
  - name: "proxmox"
    url: "https://github.com/ionos-cloud/cluster-api-provider-proxmox/releases/v0.6.2/infrastructure-components.yaml"
    type: "InfrastructureProvider"

# Configuration overrides
images:
  all:
    repository: "registry.k8s.io"
EOF
```

### Environment Variables Setup

```bash
# Create environment file
cat > ~/.capi-env << 'EOF'
# Proxmox connection settings
export PROXMOX_URL="https://192.168.0.10:8006/api2/json"
export PROXMOX_TOKEN="capi@pve!capi-token"
export PROXMOX_SECRET="12345678-1234-1234-1234-123456789abc"

# Default cluster settings
export CLUSTER_NAME="homelab-cluster"
export KUBERNETES_VERSION="v1.32.0"
export CONTROL_PLANE_MACHINE_COUNT="1"
export WORKER_MACHINE_COUNT="2"

# Network configuration
export CONTROL_PLANE_ENDPOINT_IP="192.168.0.30"
export POD_CIDR="10.244.0.0/16"
export SERVICE_CIDR="10.96.0.0/16"
EOF

# Source environment
source ~/.capi-env

# Make persistent
echo "source ~/.capi-env" >> ~/.bashrc
```

### CAPI Initialization

```bash
# Initialize Cluster API
clusterctl init \
  --infrastructure proxmox \
  --ipam in-cluster \
  --control-plane talos \
  --bootstrap talos

# Verify installation
kubectl get pods -A | grep -E "(capi|proxmox|talos)"

# Check provider status
kubectl get providers -A
```

**Expected output:**
```console
NAMESPACE                           NAME                    TYPE                    VERSION   INSTALLED
capi-bootstrap-talos-system         bootstrap-talos         BootstrapProvider       v0.6.7    True
capi-control-plane-talos-system     control-plane-talos     ControlPlaneProvider    v0.5.8    True
capi-system                         cluster-api             CoreProvider            v1.10.3   True
capx-system                         infrastructure-proxmox  InfrastructureProvider  v0.6.2    True
```

---

## Python Generator Setup e Walkthrough

### Dependencies Installation

```bash
# Create virtual environment
python3 -m venv capi-generator-env
source capi-generator-env/bin/activate

# Install dependencies
pip install jinja2 pyyaml

# Verify dependencies
python -c "import jinja2, yaml; print('Dependencies OK')"
```

### Generator Architecture Overview

Il Python generator implementa un sistema templating flessibile:

```python
# Architettura generator
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Config YAML   │───▶│  Jinja2 Template │───▶│  Cluster YAML   │
│   - Parameters  │    │  - Logic         │    │  - Resources    │
│   - Overrides   │    │  - Conditionals  │    │  - Manifests    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### Default Configuration Creation

```bash
# Generate default configuration
python cluster_generator.py --create-config homelab.yaml

# Review generated configuration
cat homelab.yaml
```

La configurazione default include tutti i parametri necessari:

```yaml
# Key sections del config file
cluster_name: "homelab-cluster"           # Nome cluster
kubernetes_version: "v1.32.0"             # Versione K8s
replicas: 1                               # Control plane nodes
allowed_nodes: ["K8S0", "K8S1", "K8S2"]   # Proxmox nodes
control_plane_endpoint:
  host: "192.168.0.30"                    # VIP address
  port: 6443                              # API port
# ... configurazioni dettagliate per VM, network, Talos
```

### Configuration Customization

#### Basic Homelab Setup

```yaml
# homelab.yaml - Configurazione per ambiente homelab tipico
cluster_name: "homelab-cluster"
namespace: "default"
kubernetes_version: "v1.32.0"
replicas: 1  # Single control plane per homelab

# Infrastructure targeting
allowed_nodes: ["K8S0"]  # Single Proxmox node
source_node: "K8S0"

# Networking (tipico homelab range)
control_plane_endpoint:
  host: "192.168.0.30"
  port: 6443

ipv4_config:
  addresses: ["192.168.0.20-192.168.0.29"]
  gateway: "192.168.0.1"  # Router homelab
  prefix: 24

# Resource allocation conservativa
machine_template:
  memory_mib: 2048  # 2GB per control plane
  cpu:
    cores: 2
  disks:
    boot_volume:
      size_gb: 20  # 20GB sufficiente per Talos

# Workers enabled per testing
workers:
  enabled: true
  replicas: 2
  machine_template:
    memory_mib: 4096  # 4GB per workers
    cpu:
      cores: 2
    disks:
      boot_volume:
        size_gb: 25
```

#### Production-Ready Setup

```yaml
# production.yaml - Configurazione per ambiente production
cluster_name: "production-cluster"
kubernetes_version: "v1.32.0"
replicas: 3  # HA control plane

# Multi-node infrastructure
allowed_nodes: ["PROD01", "PROD02", "PROD03"]

# Dedicated network segment
control_plane_endpoint:
  host: "10.0.1.100"
  port: 6443

ipv4_config:
  addresses: ["10.0.1.10-10.0.1.50"]
  gateway: "10.0.1.1"
  prefix: 24

# High-performance resources
machine_template:
  memory_mib: 4096  # 4GB per control plane
  cpu:
    cores: 4
  disks:
    boot_volume:
      size_gb: 50

# Multiple workers
workers:
  enabled: true
  replicas: 6
  machine_template:
    memory_mib: 8192  # 8GB per workers
    cpu:
      cores: 4
    disks:
      boot_volume:
        size_gb: 100  # Spazio per container images
```

### Command Line Usage Examples

#### Quick Cluster Generation

```bash
# Control plane only
python cluster_generator.py \
  --cluster-name "dev-cluster" \
  --replicas 1 \
  --workers-disabled \
  --control-plane-ip "192.168.0.31" \
  --output dev-cluster.yaml

# High availability cluster
python cluster_generator.py \
  --cluster-name "ha-cluster" \
  --replicas 3 \
  --worker-replicas 5 \
  --memory 4096 \
  --worker-memory 8192 \
  --output ha-cluster.yaml

# Resource-constrained environment
python cluster_generator.py \
  --cluster-name "minimal-cluster" \
  --memory 1536 \
  --cores 1 \
  --disk-size 15 \
  --workers-disabled \
  --output minimal.yaml
```

#### Advanced Configuration

```bash
# Multi-node deployment with specific targeting
python cluster_generator.py \
  --config production.yaml \
  --allowed-nodes "NODE01,NODE02,NODE03" \
  --replicas 3 \
  --worker-replicas 6 \
  --output production-cluster.yaml

# Custom networking
python cluster_generator.py \
  --control-plane-ip "172.16.1.100" \
  --gateway "172.16.1.1" \
  --dns-servers "172.16.1.1,8.8.8.8" \
  --output custom-network.yaml
```

### Template Logic Deep Dive

Il template Jinja2 implementa logica condizionale per supportare diverse configurazioni:

```yaml
# Esempio: Worker nodes condizionali
{%- if workers.enabled %}
---
# Worker deployment solo se abilitato
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: {{ cluster_name }}-workers
spec:
  replicas: {{ workers.replicas }}
  # ... configurazione completa
{%- endif %}
```

**Vantaggi dell'approccio templating:**
- **Configurazioni parametriche** invece di YAML statici
- **Validazione** automatica dei parametri
- **Riusabilità** tra ambienti diversi
- **Version control** delle configurazioni infrastrutturali

---

## Deploy del Primo Cluster Workload

### Pre-Deployment Validation

Prima del deployment, validare che tutti i prerequisiti siano soddisfatti:

```bash
# Verify management cluster
kubectl get nodes -o wide
kubectl get pods -A | grep -E "(capi|proxmox|talos)"

# Test Proxmox connectivity
curl -k -H "Authorization: PVEAPIToken=$PROXMOX_TOKEN=$PROXMOX_SECRET" \
     "$PROXMOX_URL/nodes" | jq '.data[].node'

# Verify template exists
curl -k -H "Authorization: PVEAPIToken=$PROXMOX_TOKEN=$PROXMOX_SECRET" \
     "$PROXMOX_URL/nodes/K8S0/qemu/8700/config" | jq '.data.template'
```

### Cluster Configuration Generation

```bash
# Generate homelab cluster configuration
python cluster_generator.py \
  --config homelab.yaml \
  --output homelab-cluster.yaml

# Review generated configuration
head -50 homelab-cluster.yaml
```

### Cluster Deployment

```bash
# Apply cluster configuration
kubectl apply -f homelab-cluster.yaml

# Verify resources created
kubectl get clusters,machines,machinedeployments -A -o wide
```

**Expected initial state:**
```console
NAME                    PHASE    AGE   VERSION
cluster/homelab-cluster           1m    

NAME                                      CLUSTER           NODENAME   PROVIDERID   PHASE      AGE   VERSION
machine/homelab-cluster-cp-abc123         homelab-cluster              proxmox://   Pending    1m    v1.32.0
```

### Deployment Monitoring

Il deployment progredisce attraverso diverse fasi. Monitorare usando:

```bash
# Watch cluster progression
watch 'kubectl get clusters,machines -A -o wide'

# Monitor events for troubleshooting
kubectl get events --sort-by='.lastTimestamp' -A | tail -20

# Check specific machine status
kubectl describe machine homelab-cluster-cp-abc123
```

#### Phase 1: Infrastructure Provisioning

```bash
# Monitor Proxmox machines
kubectl get proxmoxmachines -A -o wide

# Check VM creation in Proxmox
qm list | grep -v template
```

**Expected progression:**
1. **ProxmoxMachine** resource creato
2. **VM clone** started in Proxmox
3. **VM boot** con Talos ISO
4. **Network configuration** applied

#### Phase 2: Bootstrap Process

```bash
# Monitor bootstrap configuration
kubectl get talosconfigs -A -o wide

# Check bootstrap status
kubectl describe talosconfig homelab-cluster-cp-abc123
```

**Bootstrap activities:**
1. **Talos configuration** injection
2. **Kubernetes components** installation
3. **etcd cluster** initialization
4. **API server** startup

#### Phase 3: Control Plane Ready

```bash
# Wait for control plane ready
kubectl wait --for=condition=ControlPlaneReady cluster/homelab-cluster --timeout=20m

# Check control plane status
kubectl get taloscontrolplane -A -o wide
```

#### Phase 4: Worker Nodes (if enabled)

```bash
# Monitor worker deployment
kubectl get machinedeployment -A -o wide

# Watch worker machines
kubectl get machines -A | grep worker
```

### Troubleshooting Deployment Issues

#### Common Issues e Resolution

**1. VM Creation Failures**
```bash
# Check Proxmox machine status
kubectl describe proxmoxmachine homelab-cluster-cp-abc123

# Common causes:
# - Template not found (template_id: 8700)
# - Insufficient resources on source_node
# - Network bridge misconfiguration
```

**2. Bootstrap Failures**
```bash
# Check bootstrap configuration
kubectl get talosconfig homelab-cluster-cp-abc123 -o yaml

# Common causes:  
# - Invalid Talos configuration
# - Network connectivity issues
# - Cloud-init not working
```

**3. Control Plane Issues**
```bash
# Check control plane provider logs
kubectl logs -n capi-control-plane-talos-system deployment/capi-control-plane-talos-controller-manager

# Common causes:
# - etcd initialization failures
# - Certificate generation issues
# - API server startup problems
```

### Successful Deployment Verification

Una volta completato il deployment:

```bash
# Verify cluster is ready
kubectl get cluster homelab-cluster -o wide
# Expected: PHASE=Provisioned, CONTROLPLANE=true, INFRASTRUCTURE=true

# Check all machines running
kubectl get machines -A -o wide
# Expected: All machines in "Running" phase

# Verify control plane endpoint
curl -k https://192.168.0.30:6443/version
# Expected: Kubernetes version response
```

---

## Accesso al Cluster Workload

### Kubeconfig Extraction

```bash
# Extract kubeconfig from management cluster
kubectl get secret homelab-cluster-kubeconfig -o jsonpath='{.data.value}' | base64 -d > kubeconfig-homelab

# Test cluster access
kubectl --kubeconfig kubeconfig-homelab get nodes -o wide
```

**Expected nodes output:**
```console
NAME                        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
homelab-cluster-cp-abc123   Ready    control-plane   10m   v1.32.0   192.168.0.21  <none>
homelab-cluster-worker-xyz  Ready    <none>          8m    v1.32.0   192.168.0.22  <none>
homelab-cluster-worker-def  Ready    <none>          8m    v1.32.0   192.168.0.23  <none>
```

### Cluster Verification

```bash
# Check cluster info
kubectl --kubeconfig kubeconfig-homelab cluster-info

# Verify system pods
kubectl --kubeconfig kubeconfig-homelab get pods -A | head -10

# Check cluster components
kubectl --kubeconfig kubeconfig-homelab get componentstatuses
```

### Network Connectivity Testing

```bash
# Test pod-to-pod communication
kubectl --kubeconfig kubeconfig-homelab run test-pod --image=busybox --restart=Never -- sleep 3600

# Get pod IP
POD_IP=$(kubectl --kubeconfig kubeconfig-homelab get pod test-pod -o jsonpath='{.status.podIP}')

# Test connectivity from another pod
kubectl --kubeconfig kubeconfig-homelab run test-ping --image=busybox --restart=Never -- ping -c 3 $POD_IP
```

### CNI Installation (if needed)

Se il cluster non ha un CNI preinstallato:

```bash
# Install Calico CNI
kubectl --kubeconfig kubeconfig-homelab apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Or install Cilium
kubectl --kubeconfig kubeconfig-homelab apply -f https://raw.githubusercontent.com/cilium/cilium/1.15.0/install/kubernetes/quick-install.yaml

# Wait for CNI pods ready
kubectl --kubeconfig kubeconfig-homelab wait --for=condition=Ready pods -l k8s-app=calico-node -n kube-system --timeout=300s
```

---

## Post-Deployment Configuration

### Essential Addons Installation

#### MetalLB per LoadBalancer Services

```bash
# Install MetalLB
kubectl --kubeconfig kubeconfig-homelab apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

# Configure IP pool
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.40-192.168.0.49
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool
EOF
```

#### Ingress Controller

```bash
# Install Nginx Ingress
kubectl --kubeconfig kubeconfig-homelab apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller ready
kubectl --kubeconfig kubeconfig-homelab wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

#### Local Storage Provisioner

```bash
# Install local-path-provisioner per persistent volumes
kubectl --kubeconfig kubeconfig-homelab apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml

# Set as default storage class
kubectl --kubeconfig kubeconfig-homelab patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Security Configuration

#### RBAC Setup

```bash
# Create admin user for dashboard access
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

#### Network Policies (Optional)

```bash
# Default deny all ingress policy
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

### Application Deployment Testing

#### Deploy Test Application

```bash
# Deploy sample nginx application
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test-service
spec:
  type: LoadBalancer
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
EOF
```

#### Verify Application

```bash
# Check deployment
kubectl --kubeconfig kubeconfig-homelab get deployment nginx-test -o wide

# Check service and external IP
kubectl --kubeconfig kubeconfig-homelab get service nginx-test-service -o wide

# Test application access
EXTERNAL_IP=$(kubectl --kubeconfig kubeconfig-homelab get service nginx-test-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP
```
