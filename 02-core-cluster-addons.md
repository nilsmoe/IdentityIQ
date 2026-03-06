# 2. Core Cluster Add‑ons (Platform Before Apps)

Order here matters because higher layers depend on lower ones.

## 2.1. Ingress Controller (e.g., NGINX)

You'll need an Ingress controller for:

Argo CD UI  
Infisical UI  
Possibly Nexus UI  
Eventually IIQ (Prod + UAT URLs)

Install an Ingress controller early:

Namespace: e.g. ingress-nginx  
Node pinning: nodeSelector.env=mgmt (so it runs on node6)  
Expose via your chosen LoadBalancer/NodePort + external DNS

## 2.2. Certificate management (optional but recommended)

If you want TLS automation:

Install cert-manager (CRDs, controller).  
Integrate with your internal CA or Let's Encrypt (if reachable).

This will later issue certs for:  
argocd.company.com  
infisical.company.com  
nexus.company.com  
iiq.company.com, iiq-uat.company.com

You can skip this and manage certs manually, but cert-manager simplifies long‑term ops.

## 2.3. Monitoring & alerting

Before you trust the cluster with IIQ, you want eyes on it:

Prometheus + Grafana stack (or your preferred monitoring system).  
Node exporter / kube-state-metrics.  
Basic alerts:  
Node down  
Disk pressure  
Pod OOM  
CPU/memory thresholds on DB/app nodes

Pin monitoring to env=mgmt or let it spread as you prefer.
