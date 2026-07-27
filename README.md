## 📊 Status Cluster (Auto-généré)

[![Talos](https://img.shields.io/endpoint?url=https://kromgo.yong.ovh//badges/talos_version?format=shields&style=for-the-badge&logo=talos&logoColor=white&label=%20&color=blue)](https://www.talos.dev/)&nbsp;&nbsp;
[![Kubernetes](https://img.shields.io/endpoint?url=https://kromgo.yong.ovh//badges/kubernetes_version?format=shields&style=for-the-badge&logo=kubernetes&logoColor=white&label=%20&color=blue)](https://www.kubernetes.io/)&nbsp;&nbsp;
[![Flux](https://img.shields.io/endpoint?url=https://kromgo.yong.ovh//badges/flux_version?format=shields&style=for-the-badge&logo=flux&logoColor=white&color=blue&label=%20)](https://fluxcd.io)&nbsp;&nbsp;


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