# 3. GitOps / CI / Artifacts / Secrets

At this layer, you establish the delivery pipeline and secrets model. Do this before IIQ so you don't retrofit later.

## 3.1. Git server (if not already)

Could be GitHub Enterprise, GitLab, Bitbucket, or similar.

Create repos:  
iiq-app (SSD project: rules, workflows, build.xml, Dockerfile)  
iiq-infra (Helm charts, K8s manifests, Argo CD apps, CNPG manifests)

If Git is already in place, just ensure the repos are created and reachable.

## 3.2. Artifact Repository – Nexus

Why now? IIQ WAR and maybe JDBC drivers live here; CI pulls them during builds.

Install Nexus (either on a VM or in K8s on env=mgmt node).  
Configure repositories:  
Raw / Maven repo for identityiq.war.  
Optionally proxy repos for Maven/JARs, Docker, etc.

Create a CI user with limited credentials.

Decide if Docker registry is:  
Nexus Docker repo, or  
External container registry (Harbor, ECR, ACR, etc.)

If you'll host Docker in Nexus:

Expose Nexus Docker port via Ingress or direct NodePort/LB.  
Configure TLS.

## 3.3. CI system (Jenkins, GitHub Actions, GitLab CI, etc.)

If Ansible is ready; CI can be:

Jenkins on node6 (env=mgmt), or  
GitHub/GitLab cloud, whatever fits your org.

Key: CI must be able to:

Pull from Git.  
Fetch identityiq.war from Nexus.  
Run ant full-build.  
Build & push Docker images.  
Update Helm values in iiq-infra repo (for Argo CD to detect).

You can wire CI later, but at least plan where it will run.

## 3.4. GitOps – Argo CD

Install Argo CD now, before app/DB, so:

All subsequent workloads (CloudNativePG, IIQ, Jobs) can be managed declaratively.  
You avoid "click-ops" for real workloads.

Steps:

Install Argo CD into namespace argocd on env=mgmt node.  
Create an Ingress (e.g., argocd.company.com).  
Hook Argo CD up to your Git repos (iiq-infra, iiq-app).  
Define Argo CD Applications for:  
platform/cnpg-operator  
platform/infisical  
Later: iiq/prod, iiq/uat, iiq-db/prod, iiq-db/uat

From this point onward, install everything else via Argo CD where possible.

## 3.5. Secrets manager – Infisical

Now set up the secrets backbone:

Deploy Infisical:  
Server (if self-hosted) + operator or  
Just the operator if using Infisical Cloud.

Create Projects & Environments in Infisical:  
Project: IIQ  
Env: prod  
Env: uat  
(Optional) dev or local for non‑cluster dev.

Configure Kubernetes Secret Sync:  
prod → namespace sailpoint-prod  
uat → namespace sailpoint-uat

Plan/define:  
IIQ_DB_HOST, IIQ_DB_USER, IIQ_DB_PASSWORD, etc.  
SMTP, LDAP, any external connectors.

Now you've got secrets flowing cleanly before IIQ or DB show up.
