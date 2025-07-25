---
title: "Parte 4: Setup Pratico - Day 1 Operations"
date: 2025-07-24T10:30:00+01:00
description: Guida completa al deployment iniziale di cluster Kubernetes utilizzando Cluster API (CAPI) - Da Zero a Cluster Funzionante
menu:
  sidebar:
    name: Setup Pratico
    identifier: CAPI-4
    weight: 25
    parent: CAPI
tags: ["Kubernetes", "CAPI", "Cluster API", "Infrastructure as Code", "DevOps", "Day 1 Operations"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

# Parte 4: Setup Pratico - Day 1 Operations

*Quarto articolo della serie "Deploy Kubernetes con Cluster API: Gestione Automatizzata dei Cluster"*

---

Nelle parti precedenti abbiamo esplorato i fondamenti teorici di Cluster API, l'architettura dei componenti e l'integrazione con Talos Linux. È ora il momento di mettere in pratica questi concetti attraverso un'implementazione completa delle **Day 1 Operations**.

Questa parte guiderà attraverso ogni step del processo di deployment iniziale: dalla configurazione dell'infrastruttura Proxmox al primo cluster workload funzionante e verificato, utilizzando il Python generator per automatizzare la generazione delle configurazioni parametriche.

L'obiettivo è ottenere un cluster Kubernetes **minimalmente funzionante e verificato**, pronto per le configurazioni avanzate che verranno coperte nella Parte 5 (Day 2 Operations).

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
v1.10.5/nocloud-amd64.iso
```

Questa immagine include:
- **QEMU Guest Agent** per comunicazione VM-host
- **NoCloud datasource** per cloud-init integration
- **Optimized kernel** per virtualizzazione

#### Template VM Creation

```bash
# Create template VM
qm create 8700 \
  --name "talos-template" \
  --ostype l26 \
  --memory 2048 \
  --balloon 0 \
  --cores 2 \
  --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 \
  --scsi0 local-lvm:20,format=qcow2 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --boot order=ide2 \
  --agent enabled=1,fstrim_cloned_disks=1

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

Per facilitare la creazione dei template per i workload cluster è stato creato un [repository specifico](https://github.com/monte97/homelab-capi). Per informazioni specifiche, fare riferimento alla [relativa documentazione](https://github.com/monte97/homelab-capi/blob/master/docs.md).  

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

La configurazione default include i parametri minimi per avviare un workload cluster:

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

---

## Day 1 Readiness Validation

### Cluster Health Verification

Prima di considerare completate le Day 1 Operations, è essenziale validare che il cluster sia in uno stato healthy e pronto per le configurazioni avanzate.

```bash
# Comprehensive cluster health check
kubectl --kubeconfig kubeconfig-homelab get componentstatuses
kubectl --kubeconfig kubeconfig-homelab get nodes -o wide
kubectl --kubeconfig kubeconfig-homelab get pods -A | grep -E "(kube-system|kube-public)"

# API Server responsiveness test
kubectl --kubeconfig kubeconfig-homelab cluster-info
kubectl --kubeconfig kubeconfig-homelab api-resources --verbs=list --namespaced -o name | head -10 | xargs -n 1 kubectl --kubeconfig kubeconfig-homelab get -A
```

### Core System Validation

```bash
# Verify etcd cluster health
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -l component=etcd -o wide

# Check control plane components
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -l tier=control-plane

# Verify scheduler and controller-manager
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system | grep -E "(scheduler|controller-manager)"
```

### Basic Networking Tests

```bash
# DNS functionality test
kubectl --kubeconfig kubeconfig-homelab run dns-test --image=busybox --restart=Never -- nslookup kubernetes.default.svc.cluster.local

# Wait for pod completion and check results  
kubectl --kubeconfig kubeconfig-homelab wait --for=condition=Ready pod/dns-test --timeout=60s
kubectl --kubeconfig kubeconfig-homelab logs dns-test

# Basic external connectivity test
kubectl --kubeconfig kubeconfig-homelab run network-test --image=busybox --restart=Never -- ping -c 3 8.8.8.8
kubectl --kubeconfig kubeconfig-homelab wait --for=condition=Ready pod/network-test --timeout=60s
kubectl --kubeconfig kubeconfig-homelab logs network-test

# Cleanup test pods
kubectl --kubeconfig kubeconfig-homelab delete pod dns-test network-test
```

### Service Discovery Validation

```bash
# Test service discovery
kubectl --kubeconfig kubeconfig-homelab get svc -A

# Verify kube-dns/coredns service
kubectl --kubeconfig kubeconfig-homelab get svc -n kube-system | grep dns

# Test service endpoint resolution
kubectl --kubeconfig kubeconfig-homelab get endpoints -n kube-system
```

### Resource Availability Check

```bash
# Check node resources
kubectl --kubeconfig kubeconfig-homelab describe nodes | grep -A 5 "Allocated resources"

# Verify system resource consumption
kubectl --kubeconfig kubeconfig-homelab top nodes --kubeconfig kubeconfig-homelab 2>/dev/null || echo "Metrics server not yet available (expected)"

# Check for any resource constraints
kubectl --kubeconfig kubeconfig-homelab get events --field-selector type=Warning -A
```
