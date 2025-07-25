---
title: "Parte 2: Anatomia di Cluster API - Componenti e Meccanismi"
date: 2025-07-24T10:30:00+01:00
description: Guida completa al deployment e gestione di cluster Kubernetes utilizzando Cluster API (CAPI) per l'automazione dell'infrastruttura
menu:
  sidebar:
    name: Componenti e Meccanismi
    identifier: CAPI-2
    weight: 20
    parent: CAPI
tags: ["Kubernetes", "CAPI", "Cluster API", "Infrastructure as Code", "DevOps", "Automazione"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

## Architettura dei Componenti CAPI

Cluster API implementa un'architettura modulare basata sul [pattern dei controller Kubernetes](https://kubernetes.io/docs/concepts/architecture/controller/), dove ogni componente ha responsabilità specifiche e ben definite. Questa separazione delle responsabilità garantisce estensibilità, manutenibilità e testabilità del sistema.

### Management Cluster vs Workload Cluster

La distinzione fondamentale in CAPI è la separazione tra il cluster che gestisce l'infrastruttura e i cluster che eseguono i workload applicativi.

#### Management Cluster

Il Management Cluster serve come **hub di controllo centrale** per l'infrastruttura Kubernetes. Le sue caratteristiche principali includono:

**Responsabilità operative:**
- Hosting dei [controller CAPI core](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/)
- Archiviazione delle Custom Resource Definitions che rappresentano lo stato desiderato
- Orchestrazione del ciclo di vita dei Workload Cluster
- Gestione delle credenziali di accesso all'infrastruttura

**Requisiti tecnici:**
- Cluster Kubernetes standard (può essere locale con [kind](https://kind.sigs.k8s.io/))
- Accesso di rete ai provider di infrastruttura
- Capacità computazionale limitata (principalmente per i controller)
- High availability opzionale ma raccomandata per ambienti production

#### Workload Cluster

I Workload Cluster rappresentano i cluster Kubernetes di destinazione dove vengono deployate le applicazioni business. Caratteristiche distintive:

**Lifecycle gestito dichiarativamente:**
```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: workload-cluster-01
spec:
  controlPlaneRef:
    kind: TalosControlPlane
    name: workload-cluster-01-control-plane
  infrastructureRef:
    kind: ProxmoxCluster 
    name: workload-cluster-01-proxmox
```

Noi definiamo solamente le caratteristiche che desideriamo, in questo caso un control plane gestito tramite il controller `TalosControlPlane`, e il sistema si occupa di _come_ raggiungere questo stato.

**Isolamento operativo:**
- Nessun accesso diretto ai componenti CAPI
- Gestione tramite `kubeconfig` generato automaticamente
- Scaling e maintenance orchestrati dal Management Cluster

---

## Componenti Core di CAPI

### CAPI Core Controller

Il [Core Controller](https://github.com/kubernetes-sigs/cluster-api/tree/main/controllers) rappresenta il cervello operativo di CAPI, responsabile dell'orchestrazione di alto livello del ciclo di vita dei cluster.

#### Cluster Controller

Gestisce la risorsa `Cluster`, coordinando l'interazione tra infrastructure provider e control plane provider:

**Funzioni principali:**
- Validazione della specifica del cluster
- Orchestrazione della sequenza di provisioning
- Gestione dello stato `controlPlaneReady` e `infrastructureReady`
- Generazione del `kubeconfig` per l'accesso al workload cluster

**Reconciliation logic:**
```go
// Pseudocodice del reconciliation loop
func (r *ClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) {
    // 1. Fetch cluster object
    cluster := &clusterv1.Cluster{}
    
    // 2. Reconcile infrastructure
    if !cluster.Status.InfrastructureReady {
        return r.reconcileInfrastructure(ctx, cluster)
    }
    
    // 3. Reconcile control plane  
    if !cluster.Status.ControlPlaneReady {
        return r.reconcileControlPlane(ctx, cluster)
    }
    
    // 4. Generate kubeconfig
    return r.reconcileKubeconfig(ctx, cluster)
}
```

#### Machine Controller

Il [Machine Controller](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/machine/) gestisce il ciclo di vita delle singole istanze di calcolo che compongono il cluster:

**Responsabilità operative:**
- Coordinamento con Infrastructure Provider per provisioning VM
- Gestione del processo di bootstrap tramite Bootstrap Provider  
- Monitoring dello stato dei nodi e recovery automatico
- Implementazione delle politiche di sostituzione per oggetti Machine immutabili

**Stati del ciclo di vita:**
```yaml
status:
  phase: "Running"  # Pending, Provisioning, Provisioned, Running, Deleting, Failed
  addresses:
    - type: "InternalIP"
      address: "192.168.1.100"
  nodeRef:
    kind: "Node"
    name: "workload-cluster-01-control-plane-abc123"
```

### Bootstrap Provider

Il Bootstrap Provider traduce le configurazioni dichiarative in script di inizializzazione che trasformano macchine virtuali "nude" in nodi Kubernetes funzionali.

#### Kubeadm Bootstrap Provider

Il [provider kubeadm](https://github.com/kubernetes-sigs/cluster-api/tree/main/bootstrap/kubeadm) rappresenta l'implementazione di riferimento, utilizzando [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) per l'inizializzazione dei nodi:

**Generazione cloud-init:**
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfig
metadata:
  name: worker-node-config
spec:
  joinConfiguration:
    nodeRegistration:
      kubeletExtraArgs:
        cloud-provider: external
  preKubeadmCommands:
    - "swapoff -a"
    - "modprobe br_netfilter"
  postKubeadmCommands:
    - "kubectl label node ${HOSTNAME} node-role.kubernetes.io/worker="
```

#### Talos Bootstrap Provider

Il [Talos Bootstrap Provider](https://github.com/siderolabs/cluster-api-bootstrap-provider-talos) genera configurazioni specifiche per Talos Linux:

**Configurazione Talos:**
```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: TalosConfig  
metadata:
  name: talos-worker-config
spec:
  generateType: "join"
  talosVersion: "v1.7.0"
  configPatches:
    - op: "add"
      path: "/machine/network/interfaces"
      value:
        - interface: "eth0"
          dhcp: true
```

### Control Plane Provider

Il Control Plane Provider gestisce i componenti master del cluster Kubernetes, inclusi API server, etcd, controller manager e scheduler.

#### KubeadmControlPlane Provider

Il [KubeadmControlPlane](https://cluster-api.sigs.k8s.io/tasks/automated-machine-management/control-plane/) provider utilizza kubeadm per gestire il control plane:

**Configurazione dichiarativa:**
```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: cluster-control-plane
spec:
  replicas: 3
  version: "v1.29.0"
  kubeadmConfigSpec:
    clusterConfiguration:
      etcd:
        local:
          dataDir: "/var/lib/etcd"
      networking:
        serviceSubnet: "10.96.0.0/16"
        podSubnet: "10.244.0.0/16"
```

**Rolling update strategy:**
- Update sequenziali per mantenere quorum etcd
- Validation dei componenti prima del prossimo nodo
- Rollback automatico in caso di failure

#### TalosControlPlane Provider

Il [TalosControlPlane](https://github.com/siderolabs/cluster-api-control-plane-provider-talos) provider offre gestione nativa per Talos Linux:

**Vantaggi specifici:**
- Configurazione immutabile via API
- Upgrade atomici senza downtime
- Eliminazione di SSH e accesso shell
- Integration nativa con [Talos API](https://www.talos.dev/v1.9/reference/api/)

### Infrastructure Provider

Gli Infrastructure Provider implementano l'interfaccia tra CAPI e le piattaforme di virtualizzazione/cloud specifiche.

#### Provider Pattern

Tutti i provider implementano il [contratto standard CAPI](https://cluster-api.sigs.k8s.io/developer/providers/contracts/):

**InfraCluster Resource:**
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: ProxmoxCluster
spec:
  controlPlaneEndpoint:
    host: "192.168.1.100"
    port: 6443
  ipv4Config:
    addresses: ["192.168.1.100-192.168.1.110"]
    prefix: 24
    gateway: "192.168.1.1"
```

**InfraMachine Resource:**
```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: ProxmoxMachine
spec:
  sourceNode: "proxmox-node-01"
  templateID: 8700
  diskSize: "20G"
  memoryMiB: 4096
  numCores: 2
```

#### Proxmox Provider

Il [Cluster API Provider Proxmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox) fornisce integrazione nativa con Proxmox VE:

**Funzionalità supportate:**
- VM provisioning tramite [Proxmox API](https://pve.proxmox.com/wiki/Proxmox_VE_API)
- Template cloning e customization
- Network configuration automatica
- Storage management per dischi VM
- Integration con [cloud-init](https://cloud-init.io/) per bootstrap

---

## Custom Resource Definitions (CRDs)

Le [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) rappresentano il linguaggio dichiarativo di CAPI, permettendo di definire infrastruttura Kubernetes attraverso manifest YAML standard.

### Cluster CRD

La risorsa `Cluster` serve come **entry point dichiarativo** per definire un cluster Kubernetes completo:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
  namespace: default
spec:
  clusterNetwork:
    services:
      cidrBlocks: ["10.96.0.0/16"]
    pods:  
      cidrBlocks: ["10.244.0.0/16"]
  controlPlaneEndpoint:
    host: "192.168.1.100"
    port: 6443
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: TalosControlPlane
    name: production-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: ProxmoxCluster
    name: production-proxmox
```

**Campi critici:**
- `controlPlaneRef`: riferimento al provider che gestisce il control plane
- `infrastructureRef`: riferimento al provider di infrastruttura
- `clusterNetwork`: configurazione di rete del cluster
- `controlPlaneEndpoint`: endpoint per l'accesso all'API server

### Machine CRD

La risorsa `Machine` rappresenta **l'astrazione di una singola istanza di calcolo** destinata a diventare un nodo Kubernetes:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Machine
metadata:
  name: worker-node-01
spec:
  version: "v1.29.0"
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
      kind: TalosConfig
      name: worker-bootstrap-config
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: ProxmoxMachine
    name: worker-node-01-proxmox
status:
  phase: "Running"
  addresses:
    - type: "InternalIP"
      address: "192.168.1.101"
  nodeRef:
    kind: "Node"
    name: "worker-node-01"
```

**Principio di immutabilità:**
Le Machine sono progettate come [immutable infrastructure](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure). Modifiche alla specifica richiedono sostituzione completa dell'istanza piuttosto che update in-place.

### MachineSet CRD

`MachineSet` implementa il pattern [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) per gestire gruppi di Machine identiche:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineSet
metadata:
  name: worker-machines
spec:
  replicas: 3
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: "production-cluster"
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: "production-cluster"
    spec:
      version: "v1.29.0"
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: TalosConfig
          name: worker-bootstrap-config
```

**Funzionalità:**
- Maintain desired replica count
- Replace failed machines automatically  
- Scale up/down on demand
- Template-based machine configuration

### MachineDeployment CRD

`MachineDeployment` fornisce **update dichiarativi e rolling changes** per flotte di Machine:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: worker-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: "production-cluster"
  template:
    # Machine template specification
  strategy:
    type: "RollingUpdate"
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

**Rolling update process:**
1. Create new MachineSet with updated configuration
2. Scale new MachineSet up according to strategy
3. Scale old MachineSet down as new machines become ready
4. Delete old MachineSet when migration completes

---

## Reconciliation Loop e Control Theory

### Principi del Control Loop

CAPI implementa il [controller pattern](https://kubernetes.io/docs/concepts/architecture/controller/) di Kubernetes, basato sui principi della [teoria del controllo](https://en.wikipedia.org/wiki/Control_theory):

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Desired   │───▶│  Controller  │───▶│   Actual    │
│    State    │    │              │    │    State    │
│  (Spec)     │    │              │    │  (Status)   │
└─────────────┘    └──────────────┘    └─────────────┘
       ▲                  ▲                     │
       │                  │                     │
       │            ┌──────────────┐            │
       └────────────│  Feedback    │◀───────────┘
                    │    Loop      │
                    └──────────────┘
```

### Algoritmo di Reconciliation

Ogni controller CAPI implementa la stessa logica base:

```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Observe - Fetch current state
    obj := &v1beta1.Object{}
    if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. Analyze - Compare desired vs actual state  
    if obj.DeletionTimestamp != nil {
        return r.reconcileDelete(ctx, obj)
    }
    
    // 3. Act - Take corrective action
    return r.reconcileNormal(ctx, obj)
}
```

### Idempotenza e Error Handling

#### Principi di Idempotenza

Le operazioni CAPI sono progettate per essere [idempotenti](https://en.wikipedia.org/wiki/Idempotence), permettendo esecuzione sicura multipla:

**Esempi di operazioni idempotenti:**
- Provisioning VM: verifica esistenza prima della creazione
- Configuration update: apply solo se diversa dallo stato corrente  
- Resource cleanup: ignore "not found" errors durante la cancellazione

#### Gestione Errori e Retry

CAPI implementa strategie di retry sofisticate per gestire errori transienti:

```go
// Exponential backoff per retry
return ctrl.Result{
    Requeue: true,
    RequeueAfter: time.Duration(math.Pow(2, float64(retryCount))) * time.Second,
}, nil
```

**Categorie di errori:**
- **Transient errors**: network timeouts, temporary API unavailability
- **Configuration errors**: invalid specifications, missing resources
- **Infrastructure errors**: quota exceeded, hardware failures

---

## Flusso End-to-End: Da Manifest a Cluster

### Phase 1: Resource Creation

```bash
kubectl apply -f cluster-manifest.yaml
```

**Sequenza di eventi:**
1. **API Server**: valida e persiste le risorse nel cluster
2. **CAPI Core Controller**: rileva nuova risorsa `Cluster`
3. **Admission Controllers**: applicano policies e defaults
4. **Event Recording**: registra eventi per debugging

### Phase 2: Infrastructure Provisioning

```
Cluster Controller ──→ Infrastructure Provider ──→ Proxmox API
      │                        │                        │
      │                        ▼                        ▼
      │              ProxmoxCluster Created    VM Template Cloned
      │                        │                        │
      ▼                        ▼                        ▼
  Status Update        Infrastructure Ready      VM Started
```

**Attività del Proxmox Provider:**
- Clone VM template per ogni machine
- Configure network interfaces
- Inject cloud-init configuration
- Start VM instances
- Update ProxmoxMachine status

### Phase 3: Bootstrap Process

```
Machine Controller ──→ Bootstrap Provider ──→ TalosConfig Generation
      │                       │                        │
      │                       ▼                        ▼
      │              Cloud-Init Generated      Config Applied to VM
      │                       │                        │
      ▼                       ▼                        ▼
  Bootstrap Ready       Node Joins Cluster    Kubernetes Ready
```

**Bootstrap Provider activities:**
- Generate node-specific configuration
- Create join tokens and certificates
- Configure kubelet parameters
- Setup container runtime
- Apply security policies

### Phase 4: Control Plane Initialization

Per control plane nodes, il processo include passaggi aggiuntivi:

```yaml
# TalosControlPlane reconciliation
spec:
  replicas: 3
  version: "v1.29.0"
  
# Sequenza di inizializzazione:
# 1. First node: initialize etcd cluster
# 2. Generate cluster certificates  
# 3. Start API server, controller-manager, scheduler
# 4. Additional nodes: join existing etcd cluster
# 5. Configure load balancer endpoint
```

### Phase 5: Kubeconfig Generation

Una volta che l'API server è accessibile:

```go
// Controller genera kubeconfig
kubeconfig := &corev1.Secret{
    ObjectMeta: metav1.ObjectMeta{
        Name: fmt.Sprintf("%s-kubeconfig", cluster.Name),
        Namespace: cluster.Namespace,
    },
    Data: map[string][]byte{
        "value": kubeconfigBytes,
    },
}
```

**Contenuto kubeconfig:**
- Certificate Authority del cluster
- Client certificate per admin access
- API server endpoint configuration
- Context configuration per `kubectl`

---

## Debugging e Observability

### Monitoring Controller Health

```bash
# Check controller pod status
kubectl get pods -n capi-system
kubectl get pods -n capx-system  # Infrastructure provider
kubectl get pods -n capi-bootstrap-talos-system

# Review controller logs
kubectl logs -n capi-system deployment/capi-controller-manager
kubectl logs -n capx-system deployment/capx-controller-manager
```

### Resource Status Inspection

```bash
# Check cluster status
kubectl get cluster production-cluster -o wide

# Examine machine lifecycle
kubectl get machines -A -o wide

# Review events for troubleshooting
kubectl describe cluster production-cluster
kubectl get events --sort-by='.lastTimestamp' -A
```

### Common Debugging Patterns

**Infrastructure provisioning failures:**
```bash
# Check infrastructure resources
kubectl get proxmoxclusters,proxmoxmachines -A -o wide
kubectl describe proxmoxmachine <machine-name>

# Verify Proxmox connectivity
curl -k -H "Authorization: PVEAPIToken=$PROXMOX_TOKEN=$PROXMOX_SECRET" \
     "$PROXMOX_URL/version"
```

**Bootstrap failures:**
```bash
# Examine bootstrap configuration
kubectl get talosconfigs -A -o yaml
kubectl describe talosconfig <config-name>

# Check cloud-init logs on VM
# (requires VM console access)
tail -f /var/log/cloud-init-output.log
```

---

L'architettura modulare di CAPI, basata su controller specializzati e CRDs estensibili, fornisce un framework robusto per gestire cluster Kubernetes su qualsiasi infrastruttura. La comprensione di questi componenti e dei loro pattern di interazione è fondamentale per implementazioni production-ready.

Per approfondimenti sull'implementazione dei controller custom e l'estensione di CAPI, consultare la [Developer Guide](https://cluster-api.sigs.k8s.io/developer/guide/) e il [Kubebuilder Book](https://book.kubebuilder.io/).

*La prossima parte esplorerà Talos Linux e la sua integrazione nativa con CAPI, mostrando come l'OS immutabile semplifichi la gestione dei nodi Kubernetes.*
