# Homelab

Welcome to my Kubernetes-based Homelab repository! This repository contains the declarative GitOps configuration for my home infrastructure and services, managed via [ArgoCD](https://argo-cd.readthedocs.io/).

## 🏗️ Architecture Overview

The homelab runs a Kubernetes cluster (K3s/Kube) inside KVM/libvirt VMs on an Ubuntu host connected via Wi-Fi. It uses a dual L3 router and transparent gateway approach with custom iptables NAT routing.

- **GitOps Tool**: ArgoCD (App of Apps pattern)
- **Secrets Management**: Mozilla SOPS with age encryption
- **Ingress & Load Balancing**: MetalLB (L2 mode), Traefik, Istio
- **Storage**: Longhorn
- **Database**: CloudNativePG (PostgreSQL)

## 📁 Repository Structure

The repository is structured following GitOps principles, separating infrastructure controllers from application deployments.

- `bootstrap/` - The entry point for ArgoCD. Contains the `root-app.yaml` and the App of Apps definitions (`apps.yaml`, `infrastructure.yaml`).
- `infrastructure/` - Core cluster services, controllers, and operators.
- `apps/` - End-user applications and services running on top of the infrastructure.
- `NETWORKING.md` - Detailed explanation of the networking setup over KVM/Wi-Fi.

## 🚀 Infrastructure Components

- **[ArgoCD](infrastructure/argocd/)**: Continuous Delivery tool handling the GitOps flow.
- **[Cert-Manager](infrastructure/certs/)**: Automated TLS certificate provisioning.
- **[CloudNativePG](infrastructure/cnpg/)**: PostgreSQL operator for database management.
- **[CoreDNS](infrastructure/coredns.yaml)**: Cluster internal DNS.
- **[Istio](infrastructure/istio/)**: Service mesh.
- **[Longhorn](infrastructure/longhorn/)**: Distributed block storage for persistent volumes.
- **[MetalLB](infrastructure/metallb/)**: Bare metal load balancer.
- **[Traefik](infrastructure/traefik/)**: Kubernetes Ingress controller.
- **[Wireguard](infrastructure/wireguard/)**: Secure VPN access.

## 📦 Applications

- **[AdGuard Home](apps/adguard/)**: Network-wide ad & tracker blocking, plus local DNS.
- **[Authentik](apps/authentik/)**: Identity provider and Single Sign-On (SSO).
- **[Dashboard](apps/dashboard/)**: Homelab landing page / portal.
- **[Jellyfin](apps/jellyfin/)**: Self-hosted media server.
- **[Loki](apps/loki/)**: Log aggregation system.
- **[Monitoring](apps/monitoring/)**: Metrics collection and visualization (Prometheus & Grafana).
- **[n8n](apps/n8n/)**: Workflow automation platform.
- **[pgAdmin](apps/pgadmin/)**: PostgreSQL database administration.
- **[Bluetooth Panel](apps/bluetooth-panel/)**: Custom bluetooth management panel.
- **[Scripts](apps/scripts/)**: Automated tasks and cronjobs.

## 🔒 Secrets Management

Secrets are encrypted using [SOPS](https://github.com/getsops/sops) and `age`. 
- `.sops.yaml`: Defines creation rules and encryption keys for different environments.
- `age.agekey`: Contains the decryption key (never commit the private key!).

## 🌐 Networking & Routing

Due to IEEE 802.11 standards restricting bridged Wi-Fi interfaces, the cluster runs on an isolated virtual subnet (`192.168.122.0/24`) attached to `virbr0`. Traffic is routed from the physical host using NAT and iptables.

- See [NETWORKING.md](./NETWORKING.md) for full details on the network architecture and gateway setup.
- Use `./apply-cluster-routing.sh` to apply the necessary iptables rules for inbound traffic into the virtual network.
- **Note**: The `home-root-ca.crt` definition can be found inside `infrastructure/argocd/argocd-cm.yaml` for importing the CA into your devices.

## 🛠️ Getting Started (Bootstrapping)

1. **Install ArgoCD** on your Kubernetes cluster.
2. **Apply the root app**:
   ```bash
   kubectl apply -f bootstrap/root-app.yaml
   ```
3. ArgoCD will take over and sync the `infrastructure` and `apps` configurations, bootstrapping the entire cluster automatically.