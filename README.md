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
