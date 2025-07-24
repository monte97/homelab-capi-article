---
title: "Deploy Kubernetes con Cluster API: Gestione Automatizzata dei Cluster"
date: 2025-07-24T10:30:00+01:00
description: Guida completa al deployment e gestione di cluster Kubernetes utilizzando Cluster API (CAPI) per l'automazione dell'infrastruttura
menu:
  sidebar:
    name: Kubernetes CAPI
    identifier: kubernetes-capi-deploy
    weight: 15
tags: ["Kubernetes", "CAPI", "Cluster API", "Infrastructure as Code", "DevOps", "Automazione"]
categories: ["Kubernetes", "Cloud Native", "Infrastruttura"]
---

## 🔍 Contesto e Motivazioni

La gestione di cluster Kubernetes rappresenta una delle sfide più complesse nell'ecosistema cloud-native moderno. Man mano che il numero di nodi cresce, la complessità operativa aumenta esponenzialmente: provisioning di nuovi worker, upgrade coordinati del control plane, gestione delle configurazioni di rete, e manutenzione dell'infrastruttura sottostante diventano rapidamente ingestibili.

### La complessità della gestione tradizionale

I metodi tradizionali per la gestione dei cluster Kubernetes si basano tipicamente su:

- **Script personalizzati** per il provisioning e la configurazione dei nodi
- **Procedure manuali** documentate per upgrade e manutenzione
- **Configurazioni statiche** difficili da versionare e replicare
- **Approcci imperativi** che descrivono "come fare" piuttosto che "cosa ottenere"

Questo approccio presenta criticità significative:

- **Error-prone**: ogni intervento manuale introduce potenziali punti di fallimento
- **Time-consuming**: operazioni ripetitive che richiedono attenzione costante
- **Non riproducibile**: difficoltà nel replicare esattamente la stessa configurazione
- **Scalabilità limitata**: il carico operativo cresce linearmente con il numero di cluster

La conseguenza è che molte organizzazioni si trovano a "inseguire" la crescita della loro infrastruttura, dedicando sempre più tempo alla manutenzione invece che allo sviluppo di nuove funzionalità.

### Cluster API: Infrastructure as Code per Kubernetes

**Cluster API (CAPI)** è un sub-progetto ufficiale di Kubernetes progettato specificamente per risolvere questi problemi. Fornisce API dichiarative e tooling automatizzato per gestire l'intero ciclo di vita di cluster Kubernetes, permettendo agli utenti di definire lo stato desiderato della propria infrastruttura attraverso codice.

Il paradigma è identico a quello utilizzato per gestire le applicazioni all'interno di Kubernetes: invece di eseguire comandi imperativi, si dichiara lo stato finale desiderato e il sistema si occupa automaticamente di raggiungerlo e mantenerlo.

```yaml
# Esempio: definizione dichiarativa di un cluster
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-cluster
spec:
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: ProxmoxCluster
    name: production-proxmox
```

### Implementazione in ambiente Homelab

Per questo articolo, ho implementato Cluster API nel mio **homelab** utilizzando **Proxmox** come provider di infrastruttura. Questa combinazione rappresenta un caso d'uso realistico per molti sviluppatori e system administrator che vogliono sperimentare con tecnologie enterprise utilizzando hardware domestico.

L'obiettivo è dimostrare come CAPI possa trasformare la gestione di cluster Kubernetes da processo manuale e fragile a workflow automatizzato e riproducibile, anche in contesti non cloud-native tradizionali.

Il setup include:
- **Management Cluster**: cluster Kubernetes che ospita i controller CAPI
- **Workload Clusters**: cluster di produzione gestiti dichiarativamente
- **Proxmox Integration**: provisioning automatico di VM per i nodi Kubernetes
- **Talos Linux**: sistema operativo immutabile ottimizzato per Kubernetes

Nelle sezioni successive esploreremo i componenti core di CAPI, i meccanismi di riconciliazione e l'implementazione pratica in ambiente Proxmox.

---

## 🚀 Cluster API

### 💡 Principi Architetturali

#### Configurazione Dichiarativa e Infrastructure as Code

CAPI abbraccia completamente il **paradigma dichiarativo**, dove gli utenti definiscono lo stato desiderato dei loro cluster usando manifesti Kubernetes. Questo approccio enfatizza il specificare "**cosa**" il cluster dovrebbe essere, piuttosto che dettagliare il "**come**" costruirlo step-by-step.

Questa rappresenta un'evoluzione fondamentale dell'Infrastructure as Code (IaC), garantendo che le specifiche dell'infrastruttura siano:
- **Version-controlled**: ogni modifica è tracciata in Git
- **Auditable**: completa trasparenza sulle modifiche
- **Consistent**: stesso risultato in ambienti diversi

```console
# Approccio tradizionale (imperativo)
kubeadm init --config=/path/to/config.yaml
kubeadm join --token xyz --discovery-token-ca-cert-hash sha256:abc

# Approccio CAPI (dichiarativo)
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
spec:
  # Lo stato desiderato, non i comandi da eseguire
```

#### Integrazione GitOps per Deployment Consistenti

**La natura dichiarativa di CAPI lo rende inherentemente adatto per l'integrazione con workflow GitOps**. In un modello GitOps, i repository Git servono come singola fonte autorevole di verità per tutte le configurazioni del cluster.

Questo permette:
- **Controllo versione completo** di tutte le configurazioni
- **Auditabilità dettagliata** di tutti i cambiamenti
- **Sincronizzazione automatizzata** dello stato desiderato dal repository Git allo stato effettivo del cluster

#### Eventual Consistency

Come Kubernetes stesso, CAPI opera su un **modello di consistenza eventuale**. I suoi controller osservano continuamente lo stato corrente delle risorse e, quando rilevano discrepanze tra lo stato osservato e quello desiderato, **lavorano attivamente per riconciliare queste differenze**.

Questo design intrinseco considera i fallimenti transienti e le partizioni di rete, affidandosi a meccanismi come retry e operazioni idempotenti per raggiungere eventualmente lo stato desiderato.

### 🧩 Componenti Core

L'architettura di CAPI è composta da diversi controller specializzati che lavorano insieme per gestire i cluster Kubernetes.

#### Management Cluster vs. Workload Cluster

- **Management Cluster**: Cluster Kubernetes che serve come hub operativo per CAPI. È dove girano i componenti core di CAPI e i vari infrastructure provider, e dove sono memorizzate le Custom Resource che rappresentano lo stato desiderato di altri cluster Kubernetes.

- **Workload Cluster**: I cluster Kubernetes effettivi il cui intero ciclo di vita è gestito dal Management Cluster. Questi sono tipicamente dove vengono deployate le applicazioni e i servizi degli utenti.

#### CAPI Core Controller

**Il CAPI Core Controller agisce come orchestratore centrale del ciclo di vita del cluster**. È principalmente responsabile della gestione degli oggetti fondamentali Cluster e Machine. Questo controller avvia e supervisiona il processo di bootstrapping generale per nuovi cluster e fornisce astrazioni di livello superiore.

#### Control Plane Controller

Questo controller specializzato è esclusivamente responsabile della creazione e gestione continua dei componenti del control plane Kubernetes per il cluster gestito. Questi componenti includono tipicamente etcd, kube-apiserver, kube-controller-manager, e kube-scheduler. **Il Control Plane Controller sfrutta strumenti esistenti come kubeadm** per creare e gestire efficacemente questi componenti critici.

#### Bootstrap Controller

**Il Bootstrap Controller gioca un ruolo cruciale nel trasformare istanze di calcolo grezze in nodi Kubernetes completamente funzionali**. La sua funzione primaria è generare la configurazione necessaria, spesso sotto forma di script cloud-init o dati di bootstrap simili, che permette a una macchina di diventare un nodo Kubernetes.

#### Infrastructure Provider

Gli Infrastructure Provider sono **componenti pluggabili che servono da ponte tra gli oggetti Machine e Cluster generici di CAPI e le risorse infrastrutturali specifiche di una data piattaforma**. Traducono le richieste astratte di CAPI in azioni concrete, come il provisioning di macchine virtuali, la configurazione di reti, e l'impostazione di load balancer sull'infrastruttura sottostante.

**Il modello di provider pluggabile è una forza architettturale chiave di CAPI**. L'ambizione del progetto di gestire cluster su "qualsiasi infrastruttura" è realizzata attraverso questo design estensibile.

### Tabella: Componenti Core di Cluster API

| Componente | Responsabilità Primaria | Risorse Chiave Gestite |
|------------|------------------------|------------------------|
| **Management Cluster** | Ospita controller CAPI e provider; gestisce ciclo di vita Workload Cluster | Cluster, Machine, MachineDeployment, MachineSet |
| **CAPI Core Controller** | Orchestra ciclo di vita complessivo cluster; gestisce oggetti core Cluster e Machine | Cluster, Machine, MachineDeployment, MachineSet |
| **Control Plane Controller** | Crea e gestisce componenti control plane Kubernetes | KubeadmControlPlane, TalosControlPlane |
| **Bootstrap Controller** | Genera configurazione bootstrap nodo per trasformare macchine in nodi Kubernetes | KubeadmConfig, TalosConfig, Secrets |
| **Infrastructure Provider** | Provisioa risorse infrastrutturali sottostanti su piattaforma specifica | InfraCluster, InfraMachine |

---


## 📋 CRD (Custom Resource Definitions)

I Custom Resource Definitions (CRD) di Kubernetes sono il **meccanismo fondamentale che abilita l'estensibilità di CAPI**. Permettono agli utenti di definire e gestire nuovi tipi di risorse specifiche per l'applicazione, spesso chiamati "kinds", direttamente all'interno dell'API Kubernetes esistente.

**Questo permette a CAPI di trattare cluster Kubernetes e i loro componenti infrastrutturali sottostanti come oggetti Kubernetes di prima classe**, completamente integrati nell'API.

### Cluster CRD: Il Blueprint per un Cluster Kubernetes

'oggetto Cluster rappresenta la risorsa di livello più alto in CAPI, servendo come **blueprint dichiarativo completo per un intero cluster Kubernetes**. Incapsula configurazioni di alto livello come il layout di rete e include riferimenti alle risorse specifiche di infrastruttura e control plane che saranno utilizzate per istanziare il cluster.

**Campi Spec Chiave:**
- `clusterNetwork`: specifica range CIDR pod e service, porta API server
- `controlPlaneEndpoint`: dettagli hostname e porta per API server
- `controlPlaneRef`: riferimento all'oggetto provider-specific che definisce il control plane
- `infrastructureRef`: riferimento all'oggetto provider-specific che definisce l'infrastruttura sottostante

**Campi Status Chiave:**
- `phase`: indica la fase di attuazione corrente (Pending, Provisioning, Provisioned)
- `infrastructureReady`: denota la readiness dell'infrastruttura sottostante
- `controlPlaneReady`: indica se il control plane è pronto a ricevere richieste
- `conditions`: fornisce informazioni dettagliate sullo stato per vari componenti del cluster


### Machine CRD: Astrazione per Istanze di Calcolo

**L'oggetto Machine serve come specifica dichiarativa per un componente infrastrutturale**, come una macchina virtuale o un server bare metal, che è destinato a ospitare un nodo Kubernetes. Alla creazione di un oggetto Machine, un controller provider-specific è incaricato di provisionare l'infrastruttura sottostante corrispondente.

**Principio fondamentale**: gli oggetti Machine in CAPI sono **quasi-immutabili**. Se la specifica di una Machine viene aggiornata, **il controller non tenta di modificare l'host esistente in place. Invece, sostituisce il vecchio host con uno completamente nuovo** che corrisponde precisamente alla specifica aggiornata.


### MachineSet CRD: Gestione Gruppi di Machine Identiche

Un MachineSet è progettato per **mantenere un set stabile di Machine in esecuzione in qualsiasi momento**, operando in maniera analoga a un Kubernetes ReplicaSet. La sua funzione primaria è assicurare che un numero desiderato di machine identiche siano consistentemente in esecuzione.


### MachineDeployment CRD: Update Dichiarativi e Rolling Changes

Un MachineDeployment è responsabile di **orchestrare deployment attraverso una flotta di MachineSet**, concettualmente rispecchiando la relazione tra un Deployment Kubernetes e i suoi ReplicaSet e Pod. Fornisce update dichiarativi per Machine, gestendo abilmente lo scale-up di nuovi MachineSet e lo scale-down di quelli più vecchi durante cambi di configurazione.

**L'immutabilità dei singoli oggetti Machine guida la necessità di astrazioni di livello superiore come MachineDeployment**. Poiché gli update diretti in-place a una Machine non sono supportati, era necessario un meccanismo per gestire flotte di machine e facilitare rolling update.


### Tabella: CRD Essenziali di Cluster API

| Nome CRD | Scopo | Campi Spec Chiave | Campi Status Chiave |
|----------|-------|-------------------|---------------------|
| **Cluster** | Blueprint top-level per cluster Kubernetes | clusterNetwork, controlPlaneEndpoint, controlPlaneRef, infrastructureRef | phase, infrastructureReady, controlPlaneReady, conditions |
| **Machine** | Specifica dichiarativa per singola istanza di calcolo | version, infrastructureRef, bootstrapRef | phase, addresses, version, nodeRef, conditions |
| **MachineSet** | Mantiene set stabile di Machine identiche | replicas, selector, template | replicas, readyReplicas, availableReplicas |
| **MachineDeployment** | Fornisce update dichiarativi e scaling per Machine | replicas, selector, template, strategy | replicas, readyReplicas, updatedReplicas, unavailableReplicas |
| **KubeadmControlPlane** | Gestisce ciclo di vita control plane Kubernetes usando kubeadm | replicas, version, kubeadmConfigSpec, machineTemplate | replicas, readyReplicas, initialized, ready, version |
| **TalosControlPlane** | Gestisce ciclo di vita control plane Kubernetes usando Talos Linux | replicas, version, infrastructureTemplate, controlPlaneConfig | replicas, readyReplicas, initialized, ready, bootstrapped |


---

## ♻️ Reconciliation Loop: Osserva, Analizza, Agisci

I controller CAPI operano basandosi sul **pattern fondamentale del control loop Kubernetes**, un meccanismo di feedback continuo progettato per mantenere gli stati desiderati del sistema. In questo pattern, i controller incessantemente:

1. **Osservano** lo stato corrente delle risorse nel cluster
2. **Analizzano** eventuali discrepanze tra questo stato osservato e lo stato desiderato dichiarato
3. **Agiscono** per riconciliare queste differenze

Questo loop iterativo assicura che il cluster converga consistentemente verso la configurazione dichiarata, adattandosi ai cambiamenti e recuperando dalle deviazioni.

### Idempotenza e Gestione Errori: Operazioni Robuste

Una caratteristica critica inerente nei controller CAPI è la loro **idempotenza**. Questo principio detta che qualsiasi azione eseguita da un controller può essere ripetuta in sicurezza più volte con lo stesso input **senza produrre effetti collaterali indesiderati o divergenti**.

**L'idempotenza è fondamentale per l'automazione robusta** all'interno di CAPI. I sistemi distribuiti sono inherentemente suscettibili a frequenti fallimenti e instabilità di rete. **CAPI opera precisamente all'interno di questo ambiente distribuito, con i suoi controller che eseguono costantemente loop di riconciliazione**.

#### Gestione dello Stato Persistente

Il campo `Status` di un oggetto Kubernetes serve come **fonte autorevole di verità per lo stato osservato**. I controller non dovrebbero memorizzare stato critico nella loro memoria efimera a causa della mancanza di garanzie riguardo l'accesso parallelo attraverso multiple istanze di controller o restart delle macchine.


### Flusso End-to-End: Da kubectl apply a Cluster Provisionato

#### 1. Applicazione Manifesti CAPI al Management Cluster

Il processo inizia quando un operatore applica manifesti YAML dichiarativi al CAPI Management Cluster. Questi manifesti tipicamente includono Cluster, KubeadmControlPlane (o TalosControlPlane), MachineDeployment, e CRD specifici dell'infrastruttura.

```bash
kubectl apply -f cluster-manifests.yaml
```

#### 2. Interazioni Controller e Transizioni Stato CRD

1. **Ricezione API Server**: L'API server Kubernetes nel Management Cluster riceve la richiesta kubectl apply
2. **Controllo Ammissione**: Prima della persistenza, i controller di ammissione Kubernetes validano e potenzialmente mutano le risorse in arrivo
3. **CAPI Core Controller Osserva**: Il CAPI Core Controller rileva l'oggetto Cluster appena creato
4. **Trigger Provisioning Infrastruttura**: Basato sull'`infrastructureRef` specificato nell'oggetto Cluster, il controller Infrastructure Provider relevante viene attivato
5. **Preparazione Control Plane**: Il Control Plane Controller osserva l'oggetto Cluster e il suo `controlPlaneRef`
6. **Generazione Dati Bootstrap**: Il Bootstrap Controller genera i dati bootstrap essenziali
7. **Creazione VM con Dati Bootstrap**: L'Infrastructure Provider crea le macchine virtuali, iniettando i dati bootstrap generati

#### 3. Provisioning Infrastruttura e Bootstrap Nodi

1. **Esecuzione Bootstrap**: All'avvio, le VM appena provvisionate eseguono lo script o configurazione bootstrap iniettata
2. **Installazione Componenti Kubernetes**: Lo script procede all'installazione dei componenti Kubernetes necessari
3. **Registrazione Nodo**: I nodi appena bootstrappati si registrano con l'API server Kubernetes del workload cluster
4. **Aggiornamenti Status**: I controller CAPI monitorano continuamente lo status degli oggetti Machine
5. **Cluster Provisionato**: Una volta che tutti i componenti Kubernetes sono healthy, il workload cluster è considerato completamente provisionato


--- 

##  🐧 Talos Linux

Talos Linux rappresenta un **sistema operativo moderno, purpose-built** metticolosamente progettato per Kubernetes, integrato all'interno di CAPI mediant eun apposito infrastructure controller **[CACPPT](https://github.com/siderolabs/cluster-api-control-plane-provider-talos)**

L'integrazione di Talos Linux con CAPI crea una **potente sinergia architetturale**, portando a sicurezza migliorata e semplicità operativa attraverso immutabilità e design API-driven.

**La funzione primaria di CAPI è fornire gestione cluster dichiarativa**. Quando questo è accoppiato con Talos Linux, un sistema operativo che è inherentemente immutabile e gestito interamente via API, **il risultato è un sistema altamente coesivo**.

Questa combinazione significa che:
- **CAPI dichiara lo stato desiderato** del cluster Kubernetes
- **Talos Linux applica rigorosamente quello stato** a livello di sistema operativo
- **L'eliminazione dei metodi di configurazione manuale tradizionale** e la rimozione dell'accesso SSH ai nodi riduce significativamente la superficie di attacco

Per un enthusiast homelab, questo si traduce in una **riduzione sostanziale del tempo speso in manutenzione manuale e troubleshooting**, favorendo un'esperienza operativa più "production-like" con maggiore stabilità e affidabilità.


- **Immutabilità**: Il sistema operativo è **interamente read-only**, il che previene fondamentalmente configuration drift e modifiche non autorizzate. Questa immutabilità inherente migliora significativamente sia la sicurezza che la consistenza operativa.

- **Minimalismo**: Talos Linux include **solo i componenti assolutamente essenziali** richiesti per eseguire Kubernetes. Questo approccio minimalista riduce drasticamente la superficie di attacco e ottimizza il consumo di risorse.

- **API-driven**: Un principio di design core di Talos è la sua **gestione esclusiva via API gRPC**. Questo approccio API-centrico si allinea perfettamente con la filosofia dichiarativa e API-driven dello stesso Kubernetes. Eliminando la necessità di accesso SSH e configurazione manuale, Talos rafforza ulteriormente la sicurezza.

### TalosControlPlane (TCP) CRD

Il Custom Resource Definition **TalosControlPlane (TCP)** serve come specifica dichiarativa per un control plane Kubernetes che utilizza Talos Linux. Il suo scopo primario è **gestire l'intero ciclo di vita dei nodi control plane**, inclusi il loro bootstrapping, scaling, e eventuale cancellazione.

**Campi Spec Chiave:**
- `replicas`: specifica il count desiderato di machine control plane
- `version`: indica la versione target di Kubernetes
- `infrastructureTemplate`: riferimento alla risorsa dell'infrastructure provider
- `controlPlaneConfig`: contiene la configurazione Talos specifica
- `rolloutStrategy`: definisce come le machine control plane esistenti vengono sostituite durante gli upgrade

**Campi Status Chiave:**
- `replicas`: machine totali non-terminate
- `readyReplicas`: count di machine control plane completamente ready
- `initialized`: indica se la ConfigMap talos-config è stata caricata
- `ready`: se l'API server Talos è ready a ricevere richieste
- `bootstrapped`: se i nodi hanno ricevuto una richiesta bootstrap

### TalosConfig CRD

Il Custom Resource **TalosConfig** gioca un ruolo vitale nel customizzare la configurazione di singoli nodi Talos. Questa risorsa permette agli utenti di specificare il tipo preciso di configurazione richiesta per una data machine Talos—che sia per inizializzazione, ruolo control plane, o joining come worker node.


---

## 🎬 Demo

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Management     │    │   Proxmox VE     │    │  Workload       │
│  Cluster        │───▶│   Infrastructure │───▶│  Cluster        │
│  (Kind)         │    │                  │    │  (Talos)        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
│                                                │
├─ CAPI Controllers                              ├─ Control Plane (3x)
├─ Proxmox Provider                              ├─ Worker Nodes (3x)
├─ Talos Provider                                └─ CNI (Cilium)
└─ Bootstrap Provider                            
```