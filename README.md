# IdentityIQ
Identity IQ with GitOps continuous delivery on Kubernetes

# IdentityIQ Kubernetes Platform Playbook

## 1. Introduction & Assumptions

Here’s a concrete, opinionated **shopping list + order of operations** for your 6-node cluster before you start deploying IdentityIQ.

### Assumptions

- 6-node on-prem Kubernetes is already installed and healthy.
- Ansible is available for OS/K8s node prep and config.
- You’ll run **Prod + UAT in the same cluster** with node pinning.
- You want **GitOps (Argo CD)**, Git repos, artifact repos, and secrets handled cleanly.

We’ll break it into:

1. Node prep / cluster baseline  
2. Core cluster add-ons  
3. GitOps / CI / artifacts / secrets  
4. Database layer  
5. IdentityIQ (IIQ) application layer  

---

## 2. Node Prep / Cluster Baseline (Before Any "Tools")

You already have Ansible, so use it to **normalize the nodes**.

### 2.1 Label and (Optionally) Taint Nodes

Decide your zoning:

- Nodes 1–3: `env=prod` (Prod app + Prod DB)
- Nodes 4–5: `env=uat` (UAT app + UAT DB)
- Node 6: `env=mgmt` (Argo CD, Ingress, monitoring, etc.)

Example:

```bash
kubectl label nodes node1 node2 node3 env=prod
kubectl label nodes node4 node5 env=uat
kubectl label nodes node6 env=mgmt
```
(You can add taints later if you want stricter separation.)

### 2.2 Storage Class(es)
Before CloudNativePG and IIQ, you need:

A reliable default StorageClass for general workloads.
Ideally a fast StorageClass for Prod DB (NVMe/SSD with good IOPS).
Optionally a separate class for UAT DB.
Make sure they are present and marked appropriately (isDefaultClass where needed).

### 2.3 Cluster DNS / Time / Logging Basics
Ensure NTP is set correctly from Ansible (time skew kills clusters).
Confirm cluster DNS (CoreDNS) is healthy.
Set baseline log shipping (even if just node logs → central syslog or similar).
Once this is stable, move to tools.

### 3. Core Cluster Add-ons (Platform Before Apps)
Order here matters because higher layers depend on lower ones.

### 3.1 Ingress Controller (e.g., NGINX)
You’ll need an Ingress controller for:

- Argo CD UI
- Infisical UI
- Possibly Nexus UI
- Eventually IIQ (Prod + UAT URLs)
- Install an Ingress controller early:

Namespace: e.g. ingress-nginx
Node pinning: nodeSelector.env=mgmt (so it runs on node6)
Expose via your chosen LoadBalancer/NodePort + external DNS
3.2 Certificate Management (Optional but Recommended)
If you want TLS automation:

Install cert-manager (CRDs, controller).
Integrate with your internal CA or Let’s Encrypt (if reachable).
This will later issue certs for:

argocd.company.com
infisical.company.com
nexus.company.com
iiq.company.com, iiq-uat.company.com
You can skip this and manage certs manually, but cert-manager simplifies long‑term ops.
