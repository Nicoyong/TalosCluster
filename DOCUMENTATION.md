# Documentation - Cluster Kubernetes Talos

## 📋 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture](#architecture)
3. [Configuration Talos](#configuration-talos)
4. [Infrastructure Système](#infrastructure-système)
5. [Applications Déployées](#applications-déployées)
   - [Dashboard Kubernetes](#-dashboard-kubernetes-dashboard)
   - [Services Externes](#-services-externes-networkingexternal-services)
   - [Média](#-média-media)
   - [Travail](#-travail-work)
   - [Networking](#-networking-networking)
6. [Networking](#networking)
7. [Stockage](#stockage)
8. [Sécurité](#sécurité)
9. [Flux de Déploiement](#flux-de-déploiement)
10. [Procédures d'exploitation](#procédures-dexploitation)

---

## Vue d'ensemble

Ce projet implémente un cluster Kubernetes hautement disponible basé sur **Talos Linux** utilisant **TrueCharts ClusterTool** pour la gestion de configuration.

### Caractéristiques principales

- **OS**: Talos v1.12.2 - Un système d'exploitation Linux immuable optimisé pour Kubernetes
- **Kubernetes**: v1.35.0
- **Gestion**: GitOps avec Flux CD et Kustomize
- **Chiffrement**: SOPS pour la gestion des secrets
- **CNI**: Cilium (configuration manuelle requise)
- **Ingress**: NGINX (interne et externe)
- **Load Balancing**: MetalLB
- **Stockage**: Longhorn, OpenEBS, Snapshot Controller
- **Monitoring**: Prometheus Stack (Grafana, AlertManager)
- **Certificats**: cert-manager avec intégration Cloudflare

---

## Architecture

### Structure du projet

```
TalosCluster-main/
├── clusters/
│   └── main/
│       ├── kubernetes/           # Configurations K8s
│       │   ├── apps/            # Applications utilisateur
│       │   ├── core/            # Services système
│       │   ├── kube-system/     # Composants système K8s
│       │   ├── networking/      # Ingress et networking
│       │   └── system/          # Services système avancés
│       ├── talos/               # Configuration Talos
│       │   ├── talconfig.yaml   # Configuration cluster Talos
│       │   ├── patches/         # Patches Talos
│       │   └── generated/       # Fichiers générés (secrets)
│       └── clusterenv.yaml      # Variables d'environnement (chiffrées)
├── repositories/               # Sources Helm et Git
│   ├── helm/                   # Configurations repositories Helm
│   ├── git/                    # Repositories Git (TrueCharts, autres)
│   └── oci/                    # Repositories OCI (Flux manifests)
└── .github/
    ├── workflows/              # Actions GitHub
    └── renovate/               # Configuration Renovate (mises à jour auto)
```

### Composants principaux

```
┌─────────────────────────────────────────────────────┐
│           Talos Linux (v1.12.2)                      │
│  - Immuable, secure-by-default                      │
│  - API-driven configuration                         │
└────────────┬────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────┐
│    Kubernetes (v1.35.0) Control Plane               │
│  - k8s-control-1 (Control Plane + Scheduling)      │
└────────────┬────────────────────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼───┐        ┌───▼───┐
│Cilium │        │  VIP  │
│ (CNI) │        │(HAProxy)│
└───────┘        └───────┘
             │
┌────────────▼────────────────────────────────────────┐
│          Service Layer                              │
│  ┌─────────────────────────────────────────────┐  │
│  │ MetalLB: Load Balancing (IP Range config)  │  │
│  ├─────────────────────────────────────────────┤  │
│  │ NGINX Ingress: External & Internal         │  │
│  ├─────────────────────────────────────────────┤  │
│  │ Cert-Manager: TLS (Cloudflare DNS01)       │  │
│  └─────────────────────────────────────────────┘  │
└────────────┬────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────┐
│        Application Layer (Kustomize + Helm)         │
│                                                      │
│  Dashboard:            Système:                     │
│  ├─ Kubernetes         ├─ Blocky (DNS)            │
│  │  Dashboard          ├─ Prometheus              │
│  │                     ├─ Grafana                 │
│  Applications:         ├─ CloudNative PG          │
│  ├─ Jellyfin (Media)   ├─ Longhorn                │
│  ├─ qBittorrent        ├─ OpenEBS                 │
│  ├─ Nextcloud          └─ Spegel, VolSync, etc.   │
│  ├─ Joplin-Server      │                          │
│  ├─ RustDesk           Services Externes:         │
│  │                     ├─ AI (Intelligence)       │
│  External Services:    ├─ Ollama (LLM)            │
│  ├─ Home Assistant     ├─ Home Assistant (IoT)    │
│  ├─ Carottage          ├─ Carottage (Métier)     │
│  └─ Prusa 3D           └─ Prusa 3D (Impression)   │
└────────────┬────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────┐
│         Flux CD: GitOps Automation                   │
│  - Synchronise configurations depuis GitHub         │
│  - Renovation automatique (Renovate bot)           │
│  - Kustomize overlays et Helm releases             │
└────────────────────────────────────────────────────┘
```

---

## Configuration Talos

### talconfig.yaml - Paramètres clés

#### Cluster

```yaml
clusterName: ${CLUSTERNAME}              # Nom du cluster
talosVersion: v1.12.2                   # Version Talos
kubernetesVersion: v1.35.0              # Version Kubernetes
endpoint: https://${VIP}:6443            # Endpoint API (VIP)
allowSchedulingOnControlPlanes: true     # Planification sur control planes
```

#### Réseau

```yaml
clusterPodNets:
  - ${PODNET}                            # Réseau des pods (ex: 10.69.0.0/16)

clusterSvcNets:
  - ${SVCNET}                            # Réseau des services (ex: 10.96.0.0/12)

cniConfig:
  name: none                             # Cilium configuré manuellement
```

#### Nœud Master

```yaml
nodes:
  - hostname: k8s-control-1              # Nom du contrôle plane
    ipAddress: ${MASTER1IP_IP}           # Adresse IP statique
    controlPlane: true                   # Rôle control plane
    installDiskSelector:
      size: <= 1600GB                    # Disque d'installation
    networkInterfaces:
      - interface: eth0                  # Interface réseau primaire
        addresses:
          - ${MASTER1IP_CIDR}           # Adresse avec CIDR
        routes:
          - network: 0.0.0.0/0
            gateway: ${GATEWAY}          # Passerelle par défaut
        vip:
          ip: ${VIP}                     # VIP pour HA
```

#### Extensions et modules

**Control Plane:**
- Extensions officielles Siderolabs:
  - `util-linux-tools`
  - `iscsi-tools`
  - `qemu-guest-agent`
- Modules kernel: `nvme_tcp`, `vfio_pci`, `uio_pci_generic`
- Configuration NFS optimisée (v4.2, nconnect=16)

**Worker:**
- Mêmes extensions et modules
- Compatible iSCSI et QEMU

---

## Infrastructure Système

### Composants système déployés

#### 1. Cilium (CNI - Container Network Interface)
- **Namespace**: kube-system
- **Helm Repo**: cilium
- **Rôle**: Network plugin eBPF-based haute performance
- **Features**: Network policies, service mesh support

#### 2. MetalLB (Load Balancing)
- **Namespace**: metallb-system
- **Helm Repo**: metallb
- **Configuration**: Plage d'IPs gérée via clusterenv.yaml (`METALLB_RANGE`)
- **Mode**: L2 (Layer 2)
- **Rôle**: Assignation d'IPs externes aux services LoadBalancer

#### 3. Cert-Manager
- **Namespace**: cert-manager
- **Helm Repo**: jetstack
- **Issuers**: ClusterIssuer ACME (Let's Encrypt)
- **DNS01**: Intégration Cloudflare pour DNS challenges
- **Certificats**: Auto-renouvellement TLS

#### 4. Stockage

**Longhorn**
- **Namespace**: longhorn-system
- **Type**: Distributed block storage
- **Features**: 
  - Replicas configurables
  - Volume snapshots
  - Backups (S3-compatible)
- **Configuration S3**: Chiffrée dans clusterenv.yaml
  - `S3_URL`, `S3_ACCESSKEY`, `S3_SECRETKEY`

**OpenEBS**
- **Namespace**: openebs
- **Type**: Container-native storage
- **Storage Classes**: Complémentaire à Longhorn
- **Snapshot Controller**: Gestion centralisée des snapshots

#### 5. System Upgrade Controller
- **Gestion des mises à jour**: Talos et Kubernetes
- **Plans**: YAML dans `core/system-upgrade-controller-plans/`
- **Maintenance**: Orchestrée, zéro downtime

#### 6. Monitoring

**Kube-Prometheus-Stack**
- **Namespace**: monitoring
- **Composants**:
  - Prometheus: Collecte des métriques
  - Grafana: Visualisation dashboards
  - AlertManager: Gestion des alertes
  - Node Exporter: Métriques nœuds
- **Alertes configurées**: Disponibles dans alertmanagerconfig.yaml

**Metrics-Server**
- **Namespace**: kube-system
- **Fonction**: Fournir métriques CPU/memory pour HPA/VPA

**Node Feature Discovery**
- **Détection**: Capacités hardware des nœuds
- **Labels**: Automatique des nœuds

#### 7. Utilitaires

**Kubernetes Reflector**
- Synchronisation des secrets entre namespaces

**Spegel**
- Mise en cache distribuée des images Docker

**VolSync**
- Réplication de volumes entre clusters

**Kubelet CSR Approver**
- Auto-approbation des certificats kubelet

**Descheduler**
- Réoptimisation automatique de la distribution des pods

---

## Applications Déployées

### Organisation par catégorie

#### 📊 Dashboard (`kubernetes-dashboard/`)

**Kubernetes Dashboard**
- **Fonction**: Interface web de gestion du cluster Kubernetes
- **Type**: Application de monitoring/gestion
- **Service**: LoadBalancer avec IP fixe `DASHBOARD_IP`
- **Port**: 80 (HTTP via NGINX)
- **Accès**: https://dashboard.example.com
- **Features**: 
  - Vue d'ensemble du cluster
  - Gestion des ressources (pods, services, etc.)
  - Logs et métriques en temps réel
  - Support RBAC
- **Version**: 4.6.3 (TrueCharts)

#### 🌐 Services Externes (`networking/external-services/`)

Les services externes permettent à Kubernetes de se connecter à des services hébergés en dehors du cluster et de les exposer via Ingress.

**es-ai (Intelligence Artificielle)**
- **Fonction**: Service AI/LLM externe
- **Type**: ExternalIP Service
- **Adresse**: `${AI_IP}:${AI_PORT}`
- **Ingress**: `ai.${DOMAIN_0}` (avec certificat Let's Encrypt)
- **Intégration**: Cert-Manager avec Cloudflare DNS01
- **Cas d'usage**: Ollama, vLLM, ou autre service IA

**es-ollama (Ollama LLM)**
- **Fonction**: Serveur Ollama pour inférence de modèles LLM
- **Type**: ExternalIP Service
- **Adresse**: `${OLLAMA_IP}:${OLLAMA_PORT}`
- **Ingress**: `ollama.${DOMAIN_0}`
- **Modèles**: Configurés directement sur la machine hôte

**es-homeassistant (Home Automation)**
- **Fonction**: Intégration Home Assistant
- **Type**: ExternalIP Service
- **Adresse**: `${HOMEASSISTANT_IP}:${HOMEASSISTANT_PORT}`
- **Ingress**: `homeassistant.${DOMAIN_0}`
- **Features**: Domotique et automations

**es-carottage (Système de carottage)**
- **Fonction**: Service métier spécifique
- **Type**: ExternalIP Service
- **Adresse**: `${CAROTTAGE_IP}:${CAROTTAGE_PORT}`
- **Ingress**: `carottage.${DOMAIN_0}`

**es-prusa (Imprimante 3D)**
- **Fonction**: Interface de gestion imprimante 3D Prusa
- **Type**: ExternalIP Service
- **Adresse**: `${PRUSA_IP}:${PRUSA_PORT}`
- **Ingress**: `prusa.${DOMAIN_0}`
- **Intégration**: Monitoring et queue d'impression

---

#### 📺 Média (`media/`)

**Jellyfin**
- **Fonction**: Serveur média (photos, vidéos, musique)
- **Storage**: NFS mount depuis `JELLYFIN_NFS_PATH`
- **Accès**: via NGINX ingress
- **Persistance**: Volume Longhorn

**qBittorrent**
- **Fonction**: Téléchargement torrent
- **Réseau**: IP fixe `QBITTORRENT_IP`
- **Ports**: TCP `QBITTORRENT_TCP_PORT`, UDP `QBITTORRENT_UDP_PORT`
- **Storage**: Path `QBITTORRENT_PATH` (NFS)
- **Limitation**: Configuration réseau spéciale pour la bande passante

#### 💼 Travail (`work/`)

**Nextcloud**
- **Fonction**: Stockage cloud, synchronisation fichiers
- **Base de données**: CloudNative PostgreSQL
- **Storage**: NFS `NEXTCLOUD_NFS_PATH`
- **Authentification**: Utilisateur/mot de passe (`NEXTCLOUD_USER`, `NEXTCLOUD_PASSWORD`)
- **Features**: Contacts, calendrier, tasks

**Joplin-Server**
- **Fonction**: Prise de notes synchronisée
- **Persistance**: Base de données interne
- **Clients**: Applications desktop/mobile

#### 🌐 Networking (`networking/`)

**NGINX Ingress (Interne & Externe)**
- Deux instances: `nginx-internal` et `nginx-external`
- IPs fixes: `NGINX_INTERNAL_IP`, `NGINX_EXTERNAL_IP`
- Ingress Controller standard Kubernetes

**RustDesk**
- **Fonction**: Plateforme d'accès à distance sécurisé
- **Type**: Application de networking
- **IP**: `RUSTDESK_IP`
- **Port**: Standard 5900 (VNC-like)
- **Namespace**: networking
- **Ingress**: `rustdesk.${DOMAIN_0}` (optionnel)
- **Features**: 
  - Chiffrement end-to-end
  - Accès sans pare-feu complexe
  - Support multi-plateforme (Windows, Linux, macOS)
- **Cas d'usage**: Maintenance distante du cluster et systèmes associés

#### 🔧 Core (`core/`)

**Blocky**
- **Fonction**: Serveur DNS avec blocking (adblocking)
- **IP**: `BLOCKY_IP`
- **Features**: Résolution récursive, listes de blocage

**Kubernetes Dashboard**
- **Fonction**: Interface web de gestion K8s
- **IP**: `DASHBOARD_IP`
- **Accès**: Admin uniquement

**Cluster Issuer**
- Configuration ACME Let's Encrypt

---

## Networking

### Topologie

```
Internet
    │
    ▼
┌──────────────────────────────┐
│ NGINX External Ingress       │
│ IP: ${NGINX_EXTERNAL_IP}     │
│ (Publique - Cert TLS auto)   │
└──────────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│ MetalLB Load Balancer        │
│ Range: ${METALLB_RANGE}      │
│ (Assigne IPs aux services)   │
└──────────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│ Cilium CNI (eBPF)            │
│ Pod CIDR: ${PODNET}          │
│ Service CIDR: ${SVCNET}      │
│ Network Policies: Actives    │
└──────────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│ Services K8s                 │
│ ├─ Pods applicatifs          │
│ ├─ Services système          │
│ └─ StatefulSets persistants  │
└──────────────────────────────┘

Réseau Interne:
┌────────────────────────────────┐
│ NGINX Internal Ingress         │
│ IP: ${NGINX_INTERNAL_IP}       │
│ (pour services internes)       │
└────────────────────────────────┘
```

### Variables réseau

| Variable | Purpose | Exemple |
|----------|---------|---------|
| `VIP` | Virtual IP du control plane | 192.168.1.200 |
| `MASTER1IP_IP` | IP du nœud control plane | 192.168.1.210 |
| `MASTER1IP_CIDR` | CIDR du nœud | 192.168.1.210/24 |
| `GATEWAY` | Passerelle réseau | 192.168.1.1 |
| `PODNET` | Réseau des pods | 10.69.0.0/16 |
| `SVCNET` | Réseau des services | 10.96.0.0/12 |
| `METALLB_RANGE` | Plage IPs LoadBalancer | 192.168.1.100-192.168.1.200 |

---

## Stockage

### Architecture de stockage

```
┌─────────────────────────────────────────┐
│ Applications (PVC requests)             │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
    ┌─────────┐          ┌─────────┐
    │Longhorn │          │OpenEBS  │
    │Storage  │          │Storage  │
    │Class    │          │Class    │
    └────┬────┘          └────┬────┘
         │                    │
         ▼                    ▼
    Block Storage         Container-native
    - HA replicas         - Snapshots
    - S3 backups          - Thin provisioning
    - Snapshots
         │                    │
         └──────────┬─────────┘
                    │
                    ▼
        ┌──────────────────────┐
        │ Snapshot Controller  │
        │ (Gestion centralisée)│
        └──────────────────────┘
                    │
                    ▼
        ┌──────────────────────┐
        │ S3-compatible        │
        │ (Backups externes)   │
        │ URL: ${S3_URL}       │
        └──────────────────────┘
```

### Classes de stockage

**Longhorn (Distributed Block Storage)**
```yaml
provisioner: driver.longhorn.io
allowVolumeExpansion: true
# Replicas: 1-3 (configurable)
# Snapshots supportés
```

**OpenEBS (Container-native)**
```yaml
provisioner: openebs.io/local
# Snapshots supportés
# Performance accrue pour workloads I/O intensifs
```

### Variables de stockage chiffrées

```yaml
S3_URL:       # Endpoint S3 (ex: minio.example.com:9000)
S3_ACCESSKEY: # Access key S3
S3_SECRETKEY: # Secret key S3
S3_ENCRKEY:   # Clé de chiffrement S3 (volsync)
```

### Cas d'usage

| Application | Storage | Persistance |
|-------------|---------|-------------|
| Jellyfin | NFS + Longhorn | Mediatheque + DB |
| Nextcloud | NFS + PostgreSQL | Fichiers + DB |
| qBittorrent | NFS | Downloads |
| Joplin | PostgreSQL | Notes |
| Prometheus | Longhorn | Métriques 15j |
| Grafana | ConfigMaps | Dashboards |

---

## Sécurité

### Stratégies de sécurité

#### 1. Secrets Management

**SOPS (Secrets Operations)**
- **Chiffrement**: AGE
- **Fichier**: `.sops.yaml`
- **Clé publique**: Age recipient incluse
- **Fichiers chiffrés**: 
  - `clusterenv.yaml`
  - `clustersettings.secret.yaml`
  - `deploykey.secret.yaml`
  - Config Cilium bootstrap
  - Descheduler bootstrap

**Flux de déchiffrement**:
```
Git (SOPS chiffré)
    ↓
Flux CD lit repo
    ↓
SOPS déchiffre avec AGE (clé cluster)
    ↓
Secrets appliqués à K8s
    ↓
Applications accèdent via ServiceAccounts
```

#### 2. Certificats TLS

**Cert-Manager + Let's Encrypt**
- Issuer: ACME ClusterIssuer
- DNS Challenge: Cloudflare DNS01
- Token: `DOMAIN_0_CLOUDFLARE_TOKEN` (chiffré)
- Domaines: 
  - `DOMAIN_0`: Domaine principal
  - `DOMAIN_1`: Domaine secondaire
- Auto-renouvellement: 30 jours avant expiration

#### 3. Network Policies

**Cilium Network Policies**
- Implemented via Cilium CNI
- L3/L4 segmentation
- Pod-to-Pod communication: Explicite uniquement

#### 4. RBAC (Role-Based Access Control)

**ServiceAccounts par namespace**
- Permissions minimales
- Flux CD: Sa dédié avec permissions read/write
- System components: SAs restreintes

#### 5. SSH Public Key

**Authentification Talos**
- Fichier: `ssh-public-key.txt`
- Usage: Accès SSH aux nœuds Talos
- Protection: Git (pas de clé privée)

#### 6. Sécurité des images

**Spegel**
- Mise en cache distribuée des images
- Réduction de la surface d'attaque réseau
- Images pré-téléchargées sur chaque nœud

---

## Flux de Déploiement

### GitOps avec Flux CD

```
┌─────────────────────────────────────────────┐
│ GitHub Repository (TalosCluster)            │
│ ├─ clusters/main/kubernetes/               │
│ ├─ repositories/ (Helm sources)             │
│ └─ .github/workflows/ (CI/CD)               │
└────────────────────┬────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
    ┌──────────┐          ┌────────────┐
    │ Flux CD  │          │ Renovate   │
    │ Operator │          │ Bot        │
    │(Sync)    │          │(Auto Update│
    └────┬─────┘          │ dependencies)
         │                └────────────┘
         │
         ▼
    ┌──────────────────────┐
    │ Kustomize            │
    │ + Helm Values        │
    │ (Overlays)           │
    └────────┬─────────────┘
             │
             ▼
    ┌────────────────────────────┐
    │ Kubernetes API             │
    │ (Apply manifests)          │
    └────────┬───────────────────┘
             │
             ▼
    ┌────────────────────────────┐
    │ Controllers/Operators      │
    │ (Create resources)         │
    └────────┬───────────────────┘
             │
             ▼
    ┌────────────────────────────┐
    │ Applications running       │
    │ in cluster                 │
    └────────────────────────────┘
```

### Repositories configurés

#### Helm Repositories

| Nom | URL | Usage |
|-----|-----|-------|
| `bjw-s` | Bjw-s | Charts génériques |
| `cilium` | Cilium | Cilium CNI |
| `jetstack` | Jetstack | Cert-Manager |
| `metallb` | MetalLB | Load Balancer |
| `prometheus-community` | Prometheus | Monitoring stack |
| `node-feature-discovery` | NFD | Node labeling |
| `openebs` | OpenEBS | Storage |
| `truecharts` | TrueCharts | Apps |
| `home-ops-mirror` | Home Ops | Mirror local |
| `cloudnative-pg` | CNPG | PostgreSQL |
| `spegel` | Spegel | Image cache |

#### Git Repositories

| Nom | URL | Usage |
|-----|-----|-------|
| `this-repo` | Repo courant | Configuration cluster |
| `truecharts` | TrueCharts | Charts applications |

#### OCI Repositories

| Nom | Usage |
|-----|-------|
| `flux-manifests` | Flux CD manifests |
| `tuppr` | TuPPR charts |

### Flux Bootstrap

Le fichier `flux/bootstrap.yaml.ct` (template SOPS):
- Crée les ressources Flux initiales
- Configure GitRepository source
- Setup Kustomization pour sync
- Utilise variables du clusterenv.yaml

### Kustomize Overlays

```
kubernetes/
├── flux-entry.yaml (Entry point)
├── kustomization.yaml (Fusion)
├── apps/
│   ├── kustomization.yaml
│   ├── media/
│   ├── networking/
│   └── work/
├── core/
│   └── kustomization.yaml
├── kube-system/
│   └── kustomization.yaml
└── system/
    └── kustomization.yaml
```

### Upgrade Settings

Variables de mise à jour chiffrées:
- Stratégie auto-upgrade pour Talos
- Stratégie auto-upgrade pour Kubernetes
- Configuration dans `flux/upgradesettings.yaml`

### Renovate Configuration

Fichiers: `.github/renovate/`
- `special_talos.json5`: Règles Talos updates
- `special_longhorn.json5`: Longhorn versioning
- `special_stopdigistpinning.json5`: Digest management
- `major_dashboardapproval_disable.json5`: Dashboard major versions

---

## Procédures d'exploitation

### 1. Initialisation du Cluster

#### Pré-requis
- Un nœud avec Talos Linux bootable
- VIP réservée sur le réseau
- Variables d'environnement configurées

#### Étapes

1. **Générer les secrets Talos**
```bash
# Utiliser talosctl pour générer les fichiers de configuration
talosctl gen config TalosCluster https://VIP:6443 --output-dir ./clusters/main/talos/generated
```

2. **Appliquer la configuration**
```bash
# Depuis le nœud bootable
talosctl apply-config --insecure --nodes MASTER1IP --file talconfig.yaml
```

3. **Attendre le bootstrap**
```bash
# Attendre la disponibilité de l'API
talosctl -n MASTER1IP health
```

4. **Récupérer le kubeconfig**
```bash
talosctl kubeconfig -n MASTER1IP
```

5. **Installer Flux CD**
```bash
flux bootstrap github \
  --owner=username \
  --repo=TalosCluster \
  --branch=main \
  --path=clusters/main
```

### 2. Gestion des Secrets SOPS

#### Créer une clé AGE
```bash
age-keygen -o age-key.txt
# La clé publique va dans GITHUB_REPOSITORY/secrets
```

#### Éditer des secrets chiffrés
```bash
# Configuration .sops.yaml doit pointer la clé
sops clusters/main/clusterenv.yaml  # Édite et re-chiffre
```

#### Déployer une nouvelle clé
```bash
# Lorsque la clé change
kubectl -n flux-system patch secret sops-age \
  --type merge -p '{"data":{"age.agekey":"NEW_BASE64_KEY"}}'
```

### 3. Déploiement d'une nouvelle application

#### Via Helm Release

1. Créer structure:
```bash
mkdir -p clusters/main/kubernetes/apps/category/myapp/app
```

2. Créer `helm-release.yaml`:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: default
spec:
  chart:
    spec:
      chart: myapp
      sourceRef:
        kind: HelmRepository
        name: myrepo
  values:
    image:
      tag: latest
```

3. Créer `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helm-release.yaml
```

4. Créer Flux KS dans `ks.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v2
kind: Kustomization
metadata:
  name: myapp
spec:
  sourceRef:
    kind: GitRepository
    name: this-repo
  path: ./clusters/main/kubernetes/apps/category/myapp
```

5. Commit et push → Flux synchronisera automatiquement

### 4. Gestion des mises à jour

#### Mises à jour Talos/Kubernetes

Affichées dans `system-upgrade-controller-plans`:
- Fichiers YAML: `talos.yaml` et `kubernetes.yaml`
- Déclenche automatiquement lors du schedule

#### Mises à jour d'images (Renovate)

- Renovate scanne automatiquement `*.yaml`
- Crée des PR avec updates disponibles
- Auto-merge pour les patch versions
- Manuel pour les major versions

### 5. Accès aux applications principales

#### Accéder au Kubernetes Dashboard

```bash
# Le dashboard est accessible via LoadBalancer
# URL: http://<DASHBOARD_IP>
# Via ingress: https://dashboard.example.com

# Vérifier le statut du deployment
kubectl get deployment -n kubernetes-dashboard

# Logs du dashboard
kubectl logs -n kubernetes-dashboard -l app=kubernetes-dashboard
```

#### Gérer les Services Externes

Les services externes connectent Kubernetes à des services externes (hébergés ailleurs).

```bash
# Lister tous les services externes
kubectl get svc -n external-services

# Vérifier la configuration d'un service
kubectl get externalservice es-ai -n external-services -o yaml

# Afficher les ingress exposés
kubectl get ingress -n external-services
```

**Configuration des Services Externes dans clusterenv.yaml:**
```yaml
# Variables à configurer
AI_IP: 192.168.x.x
AI_PORT: 5000

OLLAMA_IP: 192.168.x.x
OLLAMA_PORT: 11434

HOMEASSISTANT_IP: 192.168.x.x
HOMEASSISTANT_PORT: 8123

CAROTTAGE_IP: 192.168.x.x
CAROTTAGE_PORT: 8080

PRUSA_IP: 192.168.x.x
PRUSA_PORT: 80

DASHBOARD_IP: 10.x.x.x  # IP LoadBalancer interne
DOMAIN_0: example.com
```

#### Dépannage des Services Externes

```bash
# Vérifier la connectivité vers un service externe
kubectl exec -it <pod-name> -n external-services -- ping <SERVICE_IP>

# Logs d'un service externe
kubectl logs -n external-services -l app=es-ai

# Vérifier le certificat Let's Encrypt
kubectl get certificate -n external-services
kubectl describe certificate -n external-services

# Forcer le renouvellement du certificat
kubectl annotate certificate -n external-services \
  es-ai-example-com cert-manager.io/issue-temporary-certificate="true"
```

### 5b. Monitoring et Observabilité

#### Accéder à Grafana
```bash
# Port-forward vers le dashboard
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# URL: http://localhost:3000
# Credentials: admin / prom-operator (par défaut)
```

#### Alertes Prometheus
- Configuration: `alertmanagerconfig.yaml`
- Destinations: Email, Slack, Webhook
- Requêtes: PromQL sur métriques collectées

#### Logs et Events
```bash
# Logs d'une application
kubectl logs -n namespace pod-name

# Events du cluster
kubectl get events -A --sort-by='.lastTimestamp'

# Talos logs (via talosctl)
talosctl -n MASTER1IP logs kubelet
```

### 6. Sauvegarde et Restauration

#### Stratégie de backup

**Volsync** gère:
- Réplication de volumes Longhorn
- Snapshots automatiques
- Synchronisation S3

Configuration:
```yaml
S3_URL:       # Endpoint
S3_ACCESSKEY: # Creds
S3_SECRETKEY:
S3_ENCRKEY:   # Chiffrement
```

**Procédure de backup**:
```bash
# Via Longhorn UI ou CLI
# Snapshots automatiques configurables
# Backups S3 incrémentaux
```

### 7. Dépannage courant

#### Le cluster ne démarre pas

```bash
# Vérifier l'état des nœuds
talosctl -n MASTER1IP health

# Vérifier les logs d'initialisation
talosctl -n MASTER1IP logs kubelet

# Réinitialiser si nécessaire (WARNING: data loss)
talosctl -n MASTER1IP reset --wipe
```

#### Les applications ne se lancent pas

```bash
# Vérifier les ressources Flux
kubectl get GitRepository,Kustomization -A

# Logs Flux
kubectl logs -n flux-system -l app=flux-cli

# Ressources CNPG
kubectl get PostgreSQL -A
```

#### Problèmes de réseau

```bash
# Tester Cilium
kubectl exec -n kube-system -it <cilium-pod> -- cilium status

# Policies appliquées
kubectl get networkpolicies -A

# MetalLB status
kubectl get IPAddressPool -n metallb-system
```

#### Certificats expirés

```bash
# Vérifier les certificats
kubectl get certificate -A

# Forcer le renouvellement
kubectl delete certificate -n ingress-nginx cert-name
```

---

## 📊 Résumé des ressources

### Services système installés

| Service | Version | Namespace | Storage | HA |
|---------|---------|-----------|---------|-----|
| Cilium | Latest | kube-system | Non | Oui |
| MetalLB | Latest | metallb-system | Non | Oui |
| Cert-Manager | Latest | cert-manager | Non | Oui |
| Prometheus | Latest | monitoring | Longhorn | Oui |
| Grafana | Latest | monitoring | Longhorn | Oui |
| Longhorn | Latest | longhorn-system | Bloc dist. | Oui |
| OpenEBS | Latest | openebs | Container-native | Oui |
| CloudNative PG | Latest | cnpg-system | Longhorn | Oui |
| Blocky | Latest | dns | ConfigMap | Non |
| NGINX Ingress | Latest | ingress-nginx | Non | Oui |
| **K8s Dashboard** | **4.6.3** | **kubernetes-dashboard** | **Non** | **Non** |
| **External Services** | **Latest** | **external-services** | **Non** | **Non** |

### Ressources réseau

- **Pods**: ~80-100 au démarrage complet
- **Services**: ~30-40
- **Ingress**: ~15-20 (dépend des apps)
- **PersistentVolumes**: ~10-15
- **NetworkPolicies**: 5+ (Cilium)

### Coûts ressources (approximatif)

| Component | CPU | Memory |
|-----------|-----|--------|
| Talos OS | 100m | 256Mi |
| Kubernetes | 500m | 1Gi |
| CNI (Cilium) | 200m | 512Mi |
| Storage (Longhorn) | 300m | 512Mi |
| Monitoring | 400m | 2Gi |
| Applications | Variable | Variable |
| **Total minimum** | **~1.5** | **~4.5Gi** |

---

## 🔗 Ressources externes

- [Talos Linux Documentation](https://www.talos.dev/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Flux CD Documentation](https://fluxcd.io/)
- [Cilium Documentation](https://cilium.io/)
- [Longhorn Documentation](https://longhorn.io/)
- [TrueCharts Repository](https://github.com/truecharts/charts)

---

## 📝 Notes de maintenance

### Vérifications régulières

**Hebdomadaire**:
- État des nœuds: `kubectl get nodes`
- PVC utilisés: `kubectl get pvc -A`
- Alertes Prometheus

**Mensuel**:
- Certifications expiration
- Mises à jour disponibles (Renovate PRs)
- Logs d'erreur système
- Snapshots et backups

**Trimestriel**:
- Test de restauration backups
- Analyse performance
- Review des policies réseau

### Politique de mise à jour

- **Patch versions** (1.2.x → 1.2.y): Auto via Renovate
- **Minor versions** (1.x.0 → 1.y.0): Validation manuelle
- **Major versions** (1.x → 2.x): Planification complète
- **Talos/K8s**: Coordonné via System Upgrade Controller

---

---

## 📝 Historique des modifications

### Récentes modifications (Juillet 2026)

#### ✅ Applications ajoutées/documentées
- **Kubernetes Dashboard** v4.6.3
  - Interface web de gestion du cluster
  - Accessible via LoadBalancer sur IP statique
  - Gestion centralisée des ressources K8s

- **Services Externes** (5 services)
  - **es-ai**: Service IA/LLM générique
  - **es-ollama**: Serveur Ollama pour inférence LLM
  - **es-homeassistant**: Plateforme domotique
  - **es-carottage**: Service métier dédié
  - **es-prusa**: Interface imprimante 3D
  - Tous exposés via Ingress avec certificats Let's Encrypt

#### 📦 Applications complètement déployées
| Catégorie | Applications | Status |
|-----------|--------------|--------|
| Dashboard | Kubernetes Dashboard | ✅ Documentée |
| Système | Cilium, MetalLB, Cert-Manager, Prometheus, Grafana, Longhorn, OpenEBS | ✅ Documentée |
| Média | Jellyfin, qBittorrent | ✅ Documentée |
| Travail | Nextcloud, Joplin-Server | ✅ Documentée |
| Networking | RustDesk, Services Externes (5x) | ✅ Documentée |
| Infrastructure | Flux CD, SOPS, System Upgrade, Monitoring | ✅ Documentée |

#### 🔄 Considérations pour les futures updates
- Évaluer besoin d'additional LLM services
- Potentiel scale des services externes
- Monitoring dédié pour services externes
- Stratégie backup pour configurations externes

---

**Document généré**: Juillet 2026
**Version cluster**: Talos v1.12.2, K8s v1.35.0
**Dernière mise à jour**: Juillet 2026 - Ajout Dashboard et Services Externes