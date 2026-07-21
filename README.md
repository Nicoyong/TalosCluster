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
- **Monitoring:** Prometheus + Grafana
- **Applications:** Jellyfin, Nextcloud, qBittorrent, RustDesk, Blocky DNS, etc.

## Documentation

Voir [DOCUMENTATION.md](./DOCUMENTATION.md) pour les détails complets.
