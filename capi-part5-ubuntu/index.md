---
title: "CAPI Parte 5: Ubuntu su Proxmox - Image Builder e Deploy"
date: 2025-10-23T14:00:00+01:00
description: Guida completa al deployment di cluster Kubernetes su Proxmox utilizzando Ubuntu, Image Builder e Cluster API - Dalla creazione dell'immagine al cluster funzionante
menu:
  sidebar:
    name: Ubuntu su Proxmox
    identifier: CAPI-5
    weight: 30
    parent: CAPI
tags: ["Kubernetes", "CAPI", "Cluster API", "Ubuntu", "Proxmox", "Image Builder", "Homelab"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

Dopo aver esplorato Talos Linux nei precedenti articoli, è tempo di cambiare approccio e utilizzare Ubuntu come sistema operativo base per i nostri nodi Kubernetes. In questo post vedremo come creare un cluster Kubernetes su Proxmox utilizzando Cluster API (CAPI) e le immagini ufficiali generate con image-builder.

## Perché Ubuntu invece di Talos?

Mentre Talos offre un'esperienza immutabile e minimale eccellente, Ubuntu fornisce:
- Maggiore flessibilità per debugging e troubleshooting
- Familiarità per chi proviene da ambienti Linux tradizionali
- Supporto più ampio di tool e software
- Possibilità di personalizzazione avanzata

## Prerequisiti

Prima di iniziare, assicurati di avere:
- Un cluster Proxmox funzionante
- Un server DHCP configurato sulla rete dove verranno create le VM
- `git`, `make` e altri tool di base installati sulla tua macchina locale
- Accesso con privilegi sufficienti al cluster Proxmox

## Image Builder: Creazione dell'Immagine Base

[Image-builder](https://github.com/kubernetes-sigs/image-builder) è un progetto della community Kubernetes che si occupa di creare immagini "Kubernetes-conformant" in modo consistente tra diverse infrastrutture e provider.

### Obiettivi di Image-builder

Il progetto si pone diversi obiettivi:
- Installare tutto il software e i tool necessari per i provider downstream come Cluster API
- Supportare la customizzazione delle immagini da parte degli utenti
- Fornire un processo di build ripetibile e affidabile
- Garantire che le immagini siano sempre costruite attraverso un set noto e verificato di fasi

### Setup Iniziale

Cloniamo il repository e installiamo le dipendenze:

```bash
git clone https://github.com/kubernetes-sigs/image-builder.git
cd image-builder/images/capi

# Installa le dipendenze generali
make deps

# Installa le dipendenze specifiche per Proxmox
make deps-proxmox
```

**Nota**: Controlla l'output dei comandi `make deps` per correggere eventuali errori prima di procedere.

### Configurazione Utente e Permessi su Proxmox

Per permettere a image-builder di operare su Proxmox, dobbiamo creare un utente dedicato con i permessi appropriati.

I permessi necessari sono:
- `Datastore.*`
- `SDN.*`
- `Sys.AccessNetwork`
- `Sys.Audit`
- `VM.*`

Questi devono essere assegnati alla root path `/` e propagati.

Ecco la procedura completa per creare utente, token e ruolo dedicato:

```bash
# Crea l'utente e il token
pveum user add image-builder@pve
pveum user token add image-builder@pve builder --privsep 0

# Crea un ruolo dedicato con i permessi granulari necessari
pveum role add PackerBuilder -privs "\
VM.Allocate,\
VM.Audit,\
VM.Clone,\
VM.Config.CDROM,\
VM.Config.CPU,\
VM.Config.Cloudinit,\
VM.Config.Disk,\
VM.Config.HWType,\
VM.Config.Memory,\
VM.Config.Network,\
VM.Config.Options,\
VM.Console,\
VM.Monitor,\
VM.PowerMgmt,\
Datastore.Allocate,\
Datastore.AllocateSpace,\
Datastore.AllocateTemplate,\
Datastore.Audit"

# Applica il ruolo alle risorse necessarie
pveum acl modify /nodes/<NOME-NODO> -user image-builder@pve -role PackerBuilder
pveum acl modify /storage/local -user image-builder@pve -role PackerBuilder
pveum acl modify /storage/local-lvm -user image-builder@pve -role PackerBuilder
```

**Importante**: Durante la creazione del token verrà mostrato un secret. Copialo e salvalo in un luogo sicuro, poiché non sarà più accessibile in seguito.

### Variabili d'Ambiente

Configura le seguenti variabili d'ambiente sostituendo i valori placeholder con quelli del tuo ambiente:

```bash
export PROXMOX_URL="https://<PROXMOX-URI>/api2/json"
export PROXMOX_USERNAME='image-builder@pve!builder'
export PROXMOX_TOKEN="<generated-token>"
export PROXMOX_NODE="<proxmox-node>"
export PROXMOX_ISO_POOL="local"
export PROXMOX_STORAGE_POOL="local-lvm"
export PROXMOX_BRIDGE="vmbr0"
export PACKER_FLAGS="-var disk_format=raw"
```

### Build dell'Immagine

Ora possiamo procedere con la generazione dell'immagine Ubuntu 24.04:

```bash
make build-proxmox-ubuntu-2404
```

Questo processo può richiedere diversi minuti. Al termine, avrai un template Proxmox pronto per essere utilizzato con Cluster API.

## Deploy del Cluster Kubernetes con CAPI

### Setup del Management Cluster

Cluster API richiede un cluster Kubernetes esistente per gestire i cluster workload. Una volta configurate le variabili d'ambiente, utilizziamo kind per creare il cluster di gestione:

```bash
# Crea il cluster di management con kind
kind create cluster --name capi-management

# Genera una chiave SSH se non ne hai già una
ssh-keygen -t rsa -b 4096 -f ~/.ssh/capi-cluster -N ""

# Configura le variabili d'ambiente per CAPI
export PROXMOX_URL="https://<PROXMOX-URI>:8006"
export PROXMOX_TOKEN='<TOKEN-ID>=<TOKEN-SECRET>'
export PROXMOX_SOURCENODE="<proxmox-node>"

# Inizializza i provider Cluster API
clusterctl init --infrastructure proxmox --ipam in-cluster
```


### Generazione e Deploy del Cluster

Per prima cosa settiamo alcune variabili di ambiente per specificare alcuni parametri di generazione della configurazione

```bash
export CONTROL_PLANE_ENDPOINT_IP="<IP-DISPONIBILE>"
export NODE_IP_RANGES='[<START-IP>-<END-IP>]'
export GATEWAY="<GATEWAY-IP>"
export IP_PREFIX=<IP-PREFIX>
export DNS_SERVERS='[<DNS-SERVER>]'
export TEMPLATE_VMID="<VMID-DEL-TEMPLATE-CREATO>"
export BRIDGE="vmbr0"
export ALLOWED_NODES="[<ALLOWED_NODE1>,<ALLOWED_NODE2>]"
export BOOT_VOLUME_DEVICE="scsi0"
export BOOT_VOLUME_SIZE="15"
export NUM_SOCKETS="1"
export NUM_CORES="2"
export MEMORY_MIB="2048"
```

Generiamo il manifest del cluster:

```bash
clusterctl generate cluster capi-cluster \
  --kubernetes-version v1.32.0 \
  --control-plane-machine-count=1 \
  --worker-machine-count=2 \
  > capi-cluster.yaml
```

Prima di applicare il manifest, è consigliabile verificarlo:

```bash
# Rivedi la configurazione
cat capi-cluster.yaml

# Applica quando sei pronto
kubectl apply -f capi-cluster.yaml
```

### Monitoraggio del Deploy

Puoi monitorare lo stato del cluster con:

```bash
# Stato generale del cluster
kubectl get clusters

# Dettagli delle macchine
kubectl get machines

# Stato completo
clusterctl describe cluster capi-cluster
```

### Accesso al Cluster

Una volta che il cluster è pronto, recupera il kubeconfig:

```bash
clusterctl get kubeconfig capi-cluster > capi-cluster.kubeconfig

# Verifica l'accesso
kubectl --kubeconfig=./capi-cluster.kubeconfig get nodes
```

### Installazione del CNI

Il cluster non sarà completamente funzionale senza un plugin di rete. Installiamo Calico:

```bash
kubectl --kubeconfig=./capi-cluster.kubeconfig apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Dopo qualche minuto, i nodi dovrebbero passare allo stato `Ready`:

```bash
kubectl --kubeconfig=./capi-cluster.kubeconfig get nodes
```

## Conclusioni

Abbiamo visto come creare un cluster Kubernetes su Proxmox utilizzando Ubuntu e Cluster API. Questo approccio offre:

- **Standardizzazione**: Immagini conformi agli standard Kubernetes
- **Automazione**: Deploy completamente dichiarativo
- **Ripetibilità**: Processo riproducibile per creare cluster identici
- **Flessibilità**: Ubuntu come base permette maggiore personalizzazione

### Prossimi Passi

Con il cluster funzionante, puoi:
- Configurare storage persistente con Longhorn o Rook
- Implementare un ingress controller come Traefik o NGINX
- Configurare monitoraggio con Prometheus e Grafana
- Sperimentare con GitOps usando ArgoCD o Flux

## Risorse Utili

- [Cluster API Documentation](https://cluster-api.sigs.k8s.io/)
- [Image Builder Repository](https://github.com/kubernetes-sigs/image-builder)
- [Cluster API Provider Proxmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox)
- [Calico Documentation](https://docs.tigera.io/calico/latest/about/)

---

Foto di <a href="https://unsplash.com/it/@6heinz3r?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Gabriel Heinzer</a> su <a href="https://unsplash.com/it/foto/testo-4Mw7nkQDByk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>