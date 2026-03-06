# IdentityIQ Kubernetes Platform Playbook – Overview

Here's a concrete, opinionated "shopping list + order of operations" for your 6‑node cluster before you start deploying IdentityIQ.

## Assumptions

- 6-node on‑prem Kubernetes is already installed and healthy.
- Ansible is available for OS/K8s node prep and config.
- You'll run Prod + UAT in the same cluster with node pinning.
- You want GitOps (Argo CD), Git repos, artifact repos, and secrets handled cleanly.

## Structure

This playbook is split into the following files:

1. [01-node-prep-cluster-baseline.md](01-node-prep-cluster-baseline.md) – Node prep / cluster baseline
2. [02-core-cluster-addons.md](02-core-cluster-addons.md) – Core cluster add‑ons
3. [03-gitops-ci-artifacts-secrets.md](03-gitops-ci-artifacts-secrets.md) – GitOps / CI / artifacts / secrets
4. [04-database-layer.md](04-database-layer.md) – Database layer (CloudNativePG)
5. [05-iiq-application-layer.md](05-iiq-application-layer.md) – IIQ application layer

## Recommended Installation Order (High-Level Checklist)

You can think of it as layers:

### Cluster baseline

Node labels / storage classes / NTP / basic logging.

### Core infra

Ingress Controller  
(Optional) cert-manager  
Monitoring (Prom/Grafana, alerts)

### Delivery & secrets

Git repos (iiq-app, iiq-infra)  
Nexus (artifact & maybe Docker registry)  
CI system (Jenkins/GitHub Actions/etc.)  
Argo CD  
Infisical (server/operator + secret sync)

### Database

CloudNativePG Operator  
Prod CNPG cluster (3 instances on env=prod)  
UAT CNPG cluster (2 instances on env=uat)

### IIQ

SSD project in Git, wired CI → Docker images  
Helm chart for IIQ + import Job  
Argo CD apps for IIQ Prod and UAT

---

