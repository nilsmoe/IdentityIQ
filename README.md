# IdentityIQ Kubernetes installation Plan

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


Namespace: e.g. ingress-nginx
Node pinning: nodeSelector.env=mgmt (so it runs on node6)
Expose via your chosen LoadBalancer/NodePort + external DNS

### 3.2 Certificate Management (Optional but Recommended)
If you want TLS automation:
Install cert-manager (CRDs, controller).
Integrate with your internal CA or Let’s Encrypt (if reachable).
This will later issue certs for:

argocd.company.com
infisical.company.com
nexus.company.com
iiq.company.com, iiq-uat.company.com
You can skip this and manage certs manually, but cert-manager simplifies long‑term ops.

### 3.3 Monitoring & Alerting
#### Before you trust the cluster with IIQ, you want eyes on it:
- Prometheus + Grafana stack (or your preferred monitoring system).
- Node exporter / kube-state-metrics.

#### Basic alerts:
- Node down
- Disk pressure
- Pod OOM
- CPU/memory thresholds on DB/app nodes
- Pin monitoring to env=mgmt or let it spread as you prefer.

### 4. GitOps / CI / Artifacts / Secrets
At this layer, you establish the delivery pipeline and secrets model. Do this before IIQ so you don’t retrofit later.

### 4.1 Git Server (If Not Already)
Could be GitHub Enterprise, GitLab, Bitbucket, or similar.

#### Create repos:

iiq-app (SSD project: rules, workflows, build.xml, Dockerfile)
iiq-infra (Helm charts, K8s manifests, Argo CD apps, CNPG manifests)
If Git is already in place, just ensure the repos are created and reachable.

### 4.2 Artifact Repository – Nexus
Why now? IIQ WAR and maybe JDBC drivers live here; CI pulls them during builds.

### Install Nexus (either on a VM or in K8s on env=mgmt node).
Configure repositories:
- Raw / Maven repo for identityiq.war.
- Optionally proxy repos for Maven/JARs, Docker, etc.
- Create a CI user with limited credentials.
- Decide if Docker registry is:
- Nexus Docker repo, or
- External container registry (Harbor, ECR, ACR, etc.).

#### If you’ll host Docker in Nexus:
Expose Nexus Docker port via Ingress or direct NodePort/LB.
#### Configure TLS:

### 4.3 CI System (Jenkins, GitHub Actions, GitLab CI, etc.)
#### If  Ansible is ready; CI can be:
Jenkins on node6 (env=mgmt), or
GitHub/GitLab cloud, whatever fits your org.
Key: CI must be able to:
1. Pull from Git.
2. Fetch identityiq.war from Nexus.
3. Run ant full-build.
4. Build & push Docker images.
5. Update Helm values in iiq-infra repo (for Argo CD to detect).
You can wire CI later, but at least plan where it will run.

### 4.4 GitOps – Argo CD
Install Argo CD now, before app/DB, so All subsequent workloads (CloudNativePG, IIQ, Jobs) can be managed declaratively.
You avoid “click-ops” for real workloads.
#### Steps:
1. Install Argo CD into namespace argocd on env=mgmt node.
2. Create an Ingress (e.g., argocd.company.com).
3. Hook Argo CD up to your Git repos (iiq-infra, iiq-app).
#### Define Argo CD Applications for:
- platform/cnpg-operator
- platform/infisical
- Later: iiq/prod, iiq/uat, iiq-db/prod, iiq-db/uat

From this point onward, install everything else via Argo CD where possible.

#### 4.5 Secrets Manager – Infisical
Now set up the secrets backbone:

#### Deploy Infisical:
Server (if self-hosted) + operator, or Just the operator if using Infisical Cloud.

#### Create Projects & Environments in Infisical:
Project: IIQ
Env: prod
Env: uat
(Optional) dev or local for non‑cluster dev.
#### Configure Kubernetes Secret Sync:
prod → namespace sailpoint-prod
uat → namespace sailpoint-uat

#### Plan/define variables such as:

IIQ_DB_HOST, IIQ_DB_USER, IIQ_DB_PASSWORD, etc.
SMTP, LDAP, any external connectors.
Now you’ve got secrets flowing cleanly before IIQ or DB show up.

### 5. Database Layer – CloudNativePG
With the “platform” pieces in place (Ingress, Argo CD, Infisical), now build Postgres.

5.1 Install CloudNativePG Operator (via Argo CD)
Namespace: e.g. cnpg-system.
Managed by an Argo CD Application (so upgrades and rollbacks are easy).
5.2 Create DB Clusters
Use Argo CD + manifests (or Helm) to create:

Prod DB Cluster
Namespace: sailpoint-prod
CNPG Cluster resource:
instances: 3
nodeSelector: env=prod
storageClass: fast-ssd-prod
Ensure Services:
iiq-database-prod-rw
iiq-database-prod-ro
UAT DB Cluster
Namespace: sailpoint-uat
CNPG Cluster resource:
instances: 2
nodeSelector: env=uat
storageClass: ssd-uat
Integrate with Infisical
DB credentials come from Infisical → K8s Secret.
CNPG Cluster can be configured to use those credentials or generate its own, which you then sync back.
At this point, you have HA Postgres for both Prod and UAT ready before IIQ touches anything.

6. IdentityIQ Application Layer
Now with everything else in place, you finally prepare for IIQ itself.

6.1 SSD Project in Git & CI Wiring
Refine your SSD-based project structure (as previously discussed):

build.xml
src/
config/
docker/
helm/
Set up CI pipeline to:

Check out iiq-app repo.
Fetch identityiq.war from Nexus.
Run ant full-build -Ddocker.tag=<git-sha>.
Push image to Docker registry.
Update values-prod.yaml / values-uat.yaml in iiq-infra repo.
6.2 Helm Chart for IIQ App + Import Job
In iiq-infra/helm/iiq-chart:

Deployment, Service, Ingress, HPA (if needed).
values-prod.yaml / values-uat.yaml with:
nodeSelector.env=prod / env=uat
DB connection pointing to CNPG services.
envFrom referencing Infisical-synced secrets.
Post-install/upgrade Job that:
Runs IIQ console → imports init-custom.xml.
6.3 Argo CD Applications for IIQ
Create 4 core Applications (via YAML in iiq-infra):

iiq-db-prod → CNPG cluster manifest (if not already under a platform app).
iiq-db-uat → same for UAT.
iiq-app-prod → IIQ Prod deployment using values-prod.yaml.
iiq-app-uat → IIQ UAT deployment using values-uat.yaml.
Argo CD installs them in the right order automatically when you organize app-of-apps or set dependencies.

7. Recommended Installation Order (High-Level Checklist)
Think of it as layers.

7.1 Cluster Baseline
Node labels / storage classes / NTP / basic logging.
7.2 Core Infra
Ingress Controller
(Optional) cert-manager
Monitoring (Prom/Grafana, alerts)
7.3 Delivery & Secrets
Git repos (iiq-app, iiq-infra)
Nexus (artifact & maybe Docker registry)
CI system (Jenkins/GitHub Actions/etc.)
Argo CD
Infisical (server/operator + secret sync)
7.4 Database
CloudNativePG Operator
Prod CNPG cluster (3 instances on env=prod)
UAT CNPG cluster (2 instances on env=uat)
7.5 IIQ
SSD project in Git, wired CI → Docker images
Helm chart for IIQ + import Job
Argo CD apps for IIQ Prod and UAT
