# TalosCluster

Cluster Kubernetes basé sur [Talos Linux](https://www.talos.dev/) avec GitOps (Flux CD).

**Basé sur:** [TrueCharts](https://truecharts.org/)

## Qu'est-ce que c'est?

Un cluster Kubernetes self-hosted avec:
- **OS:** Talos Linux v1.12.2 (immuable et sécurisé)
- **K8s:** Kubernetes v1.35.0
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