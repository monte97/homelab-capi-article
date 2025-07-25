---
title: "Parte 5: Gestione Avanzata e Troubleshooting"
date: 2025-07-24T10:30:00+01:00
description: Guida completa al deployment e gestione di cluster Kubernetes utilizzando Cluster API (CAPI) per l'automazione dell'infrastruttura
menu:
  sidebar:
    name: Gestione Avanzata
    identifier: CAPI-5
    weight: 30
    parent: CAPI
tags: ["Kubernetes", "CAPI", "Cluster API", "Infrastructure as Code", "DevOps", "Automazione"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

# Parte 5: Gestione Avanzata e Troubleshooting

*Quinto e ultimo articolo della serie "Deploy Kubernetes con Cluster API: Gestione Automatizzata dei Cluster"*

---

Dopo aver implementato con successo il primo cluster Kubernetes usando Cluster API e Talos Linux, è tempo di esplorare la gestione avanzata, le operazioni day-2, e le strategie di troubleshooting per ambienti production-ready.

Questa parte finale della serie copre scaling dinamico, upgrade procedures, worker node management, monitoring avanzato, e best practices operative per mantenere cluster stabili e performanti nel tempo.

---

## Worker Node Management Avanzato

### Dynamic Scaling Operations

Il vero potere di CAPI emerge nella gestione dinamica dei worker nodes, permettendo scaling declarativo basato su workload requirements.

#### Horizontal Scaling

```bash
# Scale worker nodes dinamicamente
kubectl scale machinedeployment homelab-cluster-workers --replicas=5

# Verify scaling operation
watch 'kubectl get machinedeployment,machines -A | grep worker'

# Monitor new node join process
kubectl --kubeconfig kubeconfig-homelab get nodes -w
```

#### Targeted Node Scaling

Per scaling più granulare, creare multiple MachineDeployment con caratteristiche diverse:

```yaml
# high-memory-workers.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: homelab-cluster-high-memory-workers
  namespace: default
spec:
  clusterName: homelab-cluster
  replicas: 2
  selector:
    matchLabels:
      nodepool: "high-memory"
  template:
    metadata:
      labels:
        nodepool: "high-memory"
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: TalosConfigTemplate
          name: homelab-cluster-high-memory-config
      clusterName: homelab-cluster
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: homelab-cluster-high-memory-template
      version: v1.32.0

---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
kind: ProxmoxMachineTemplate
metadata:
  name: homelab-cluster-high-memory-template
  namespace: default
spec:
  template:
    spec:
      disks:
        bootVolume:
          disk: scsi0
          sizeGb: 40
          format: qcow2
      memoryMiB: 16384  # 16GB RAM per workload memory-intensive
      network:
        default:
          bridge: vmbr0
          model: virtio
      numCores: 4
      numSockets: 1
      sourceNode: K8S0
      templateID: 8700

---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: TalosConfigTemplate
metadata:
  name: homelab-cluster-high-memory-config
  namespace: default
spec:
  template:
    spec:
      generateType: worker
      talosVersion: v1.9
      configPatches:
        - op: replace
          path: /machine/install
          value:
            disk: /dev/sda
            extensions:
              - image: ghcr.io/siderolabs/qemu-guest-agent:9.2.0
        - op: add
          path: /machine/kubelet/extraArgs
          value:
            max-pods: "200"  # Higher pod density per high-memory nodes
            kube-reserved: "cpu=200m,memory=512Mi"
            system-reserved: "cpu=200m,memory=512Mi"
        - op: add
          path: /machine/nodeLabels
          value:
            node.kubernetes.io/instance-type: "high-memory"
            workload.kubernetes.io/type: "memory-intensive"
```

```bash
# Deploy specialized node pool
kubectl apply -f high-memory-workers.yaml

# Verify node labels
kubectl --kubeconfig kubeconfig-homelab get nodes --show-labels | grep "high-memory"
```

### Node Replacement Strategies

CAPI supporta diverse strategie per sostituzione dei nodi, cruciali per maintenance e upgrade.

#### Rolling Replacement

```bash
# Configure rolling update strategy
kubectl patch machinedeployment homelab-cluster-workers --type='merge' -p='{
  "spec": {
    "strategy": {
      "type": "RollingUpdate",
      "rollingUpdate": {
        "maxUnavailable": 1,
        "maxSurge": 1
      }
    }
  }
}'

# Trigger replacement updating machine template
kubectl patch proxmoxmachinetemplate homelab-cluster-worker-template --type='merge' -p='{
  "spec": {
    "template": {
      "spec": {
        "memoryMiB": 6144
      }
    }
  }
}'
```

#### Blue-Green Deployment Style

```yaml
# blue-green-workers.yaml - Nuovo deployment per replacement completo
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: homelab-cluster-workers-v2
  namespace: default
spec:
  clusterName: homelab-cluster
  replicas: 3
  template:
    spec:
      # Configurazione aggiornata
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: TalosConfigTemplate
          name: homelab-cluster-workers-v2-config
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha1
        kind: ProxmoxMachineTemplate
        name: homelab-cluster-worker-v2-template
```

```bash
# Deploy new worker generation
kubectl apply -f blue-green-workers.yaml

# Wait for new nodes ready
kubectl --kubeconfig kubeconfig-homelab wait --for=condition=Ready nodes -l machine.cluster.x-k8s.io/deployment-name=homelab-cluster-workers-v2

# Drain and remove old workers
kubectl scale machinedeployment homelab-cluster-workers --replicas=0
kubectl delete machinedeployment homelab-cluster-workers
```

### Node Affinity e Workload Placement

#### Taints e Tolerations per Node Pools

```bash
# Add taints to specialized nodes
kubectl --kubeconfig kubeconfig-homelab taint nodes -l workload.kubernetes.io/type=memory-intensive \
  workload.kubernetes.io/type=memory-intensive:NoSchedule

# Example pod with toleration
cat <<EOF | kubectl --kubeconfig kubeconfig-homelab apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-intensive-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-intensive-app
  template:
    metadata:
      labels:
        app: memory-intensive-app  
    spec:
      tolerations:
      - key: "workload.kubernetes.io/type"
        operator: "Equal"
        value: "memory-intensive"
        effect: "NoSchedule"
      nodeSelector:
        node.kubernetes.io/instance-type: "high-memory" 
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            memory: "8Gi"
          limits:
            memory: "12Gi"
EOF
```

#### Priority Classes per Workload Criticality

```yaml
# priority-classes.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority class for critical workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority  
value: 100
globalDefault: false
description: "Low priority class for batch workloads"
```

---

## Cluster Upgrade Procedures

### Kubernetes Version Upgrades

CAPI permette upgrade dichiarativi di Kubernetes, gestendo automaticamente la complessità dell'orchestrazione.

#### Control Plane Upgrade

```bash
# Check current version
kubectl --kubeconfig kubeconfig-homelab version --short

# Update control plane version
kubectl patch taloscontrolplane homelab-cluster-cp --type='merge' -p='{
  "spec": {
    "version": "v1.32.1"
  }
}'

# Monitor upgrade progress
watch 'kubectl get taloscontrolplane,machines -A -o wide'
```

Il processo di upgrade segue questa sequenza:
1. **First control plane node**: upgrade, wait for ready
2. **Subsequent nodes**: sequential upgrade maintaining quorum
3. **Validation**: health checks post-upgrade
4. **Rollback**: automatic se upgrade fails

#### Worker Nodes Upgrade

```bash
# Update worker nodes version
kubectl patch machinedeployment homelab-cluster-workers --type='merge' -p='{
  "spec": {
    "template": {
      "spec": {
        "version": "v1.32.1"
      }
    }
  }
}'

# Monitor rolling upgrade
kubectl get machinedeployment homelab-cluster-workers -o wide -w
```

#### Upgrade Verification

```bash
# Verify all nodes upgraded
kubectl --kubeconfig kubeconfig-homelab get nodes -o wide

# Check cluster components
kubectl --kubeconfig kubeconfig-homelab get pods -n kube-system -o wide

# Run cluster validation
kubectl --kubeconfig kubeconfig-homelab run upgrade-test --image=busybox --restart=Never -- /bin/sh -c "echo 'Cluster upgrade verification successful'"
```

### Talos OS Upgrades

Talos Linux supporta atomic upgrades dell'OS, eliminando i rischi di partial updates.

#### Control Plane OS Upgrade

```yaml
# talos-upgrade-patch.yaml
- op: add
  path: /machine/install/extensions
  value:
    - image: ghcr.io/siderolabs/qemu-guest-agent:9.3.0  # Updated version

- op: replace
  path: /machine/install/image
  value: ghcr.io/siderolabs/installer:v1.9.6  # New Talos version
```

```bash
# Apply OS upgrade patch
kubectl patch taloscontrolplane homelab-cluster-cp --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/controlPlaneConfig/controlplane/strategicPatches/-",
    "value": "- |\n  - op: replace\n    path: /machine/install/image\n    value: ghcr.io/siderolabs/installer:v1.9.6"
  }
]'

# Monitor Talos upgrade
kubectl get machines -A -w
```

#### Upgrade Rollback Strategy

```bash
# In caso di problemi, rollback immediato
kubectl patch taloscontrolplane homelab-cluster-cp --type='json' -p='[
  {
    "op": "replace", 
    "path": "/spec/controlPlaneConfig/controlplane/strategicPatches/0",
    "value": "- |\n  - op: replace\n    path: /machine/install/image\n    value: ghcr.io/siderolabs/installer:v1.9.5"
  }
]'
```

### Upgrade Automation

#### Scheduled Upgrades

```yaml
# upgrade-cronjob.yaml - Automated monthly upgrades
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-upgrade-check
  namespace: default
spec:
  schedule: "0 2 1 * *"  # First day of month at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: upgrade-checker
            image: bitnami/kubectl:latest
            command:
            - /bin/bash
            - -c
            - |
              # Check for available Kubernetes upgrades
              CURRENT_VERSION=$(kubectl version --short -o json | jq -r '.serverVersion.gitVersion')
              echo "Current cluster version: $CURRENT_VERSION"
              
              # Logic per determinare se upgrade necessario
              # Integrate con external API per latest stable versions
              
          restartPolicy: OnFailure
```

---

## Monitoring e Observability Avanzata

### Cluster Metrics e Alerting

#### Prometheus Stack Deployment

```bash
# Install Prometheus Operator
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl --kubeconfig kubeconfig-homelab create namespace monitoring

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kubeconfig kubeconfig-homelab \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.persistence.