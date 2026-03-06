# 5. IdentityIQ Application Layer

Now with everything else in place, you finally prepare for IIQ itself.

## 5.1. SSD project in Git & CI wiring

Refine your SSD-based project structure (as we discussed):  
build.xml, src/, config/, docker/, helm/.

Set up CI pipeline:

Check out iiq-app repo.  
Fetch identityiq.war from Nexus.  
Run ant full-build -Ddocker.tag=<git-sha>.  
Push image to Docker registry.  
Update values-prod.yaml / values-uat.yaml in iiq-infra repo.

## 5.2. Helm chart for IIQ app + import Job

In iiq-infra/helm/iiq-chart:

Deployment, Service, Ingress, HPA (if needed).  
values-prod.yaml / values-uat.yaml with:  
nodeSelector.env=prod / env=uat  
DB connection pointing to CNPG services.  
envFrom referencing Infisical-synced secrets.

Post-install/upgrade Job that:

Runs iiq console → import init-custom.xml.

## 5.3. Argo CD Applications for IIQ

Create 4 core Applications (via YAML in iiq-infra):

iiq-db-prod → CNPG cluster manifest (if not already under a platform app).  
iiq-db-uat → same for UAT.  
iiq-app-prod → IIQ Prod deployment using values-prod.yaml.  
iiq-app-uat → IIQ UAT deployment using values-uat.yaml.

Argo CD installs them in the right order automatically when you organize app-of-apps or set dependencies.
