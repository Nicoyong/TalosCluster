## 📊 Status Cluster (Auto-généré)

[![Talos Version](https://img.shields.io/badge/Talos-v1.12.2-blue?...)]()
[![Kubernetes Version](https://img.shields.io/badge/Kubernetes-v1.35.0-326ce5?...)]()
[![Status](https://img.shields.io/badge/Status-Active-green?...)]()
[![Last Updated](https://img.shields.io/badge/Updated-2026--07--24-lightgrey?...)]()


# TalosCluster

Cluster Kubernetes basé sur [Talos Linux](https://www.talos.dev/) avec GitOps (Flux CD).

**Basé sur:** [TrueCharts](https://truecharts.org/)


## Qu'est-ce que c'est?

Un cluster Kubernetes self-hosted avec:
- **OS:** Talos Linux (immuable et sécurisé)
- **K8s:** Kubernetes 
- **Networking:** Cilium + MetalLB
- **Stockage:** Longhorn + OpenEBS
- **GitOps:** Flux CD + Kustomize
- **Monitoring:** Prometheus + Grafana + Kubernetes Dashboard
- **Applications:** 
  - Média: Jellyfin, qBittorrent
  - Travail: Nextcloud, Joplin-Server
  - Networking: RustDesk
  - Services Externes: AI, Ollama, Home Assistant, Carottage, Prusa 3D
  - Infrastructure: Blocky DNS, Cert-Manager, System Upgrade

## Documentation

Voir [DOCUMENTATION.md](./DOCUMENTATION.md) pour les détails complets.