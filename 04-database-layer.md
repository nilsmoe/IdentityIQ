# 4. Database Layer – CloudNativePG

With the "platform" pieces in place (Ingress, Argo CD, Infisical), now build Postgres.

## 4.1. Install CloudNativePG Operator (via Argo CD)

Namespace: e.g. cnpg-system.  
Managed by an Argo CD Application (so upgrades and rollbacks are easy).

## 4.2. Create DB clusters

Use Argo CD + manifests (or Helm) to create:

### Prod DB cluster

Namespace: sailpoint-prod  
CNPG Cluster resource:  
instances: 3  
nodeSelector: env=prod  
storageClass: fast-ssd-prod

Ensure Service: iiq-database-prod-rw and iiq-database-prod-ro.

### UAT DB cluster

Namespace: sailpoint-uat  
instances: 2  
nodeSelector: env=uat  
storageClass: ssd-uat

### Integrate with Infisical

DB credentials come from Infisical → K8s Secret.  
CNPG Cluster can be configured to use those credentials or generate its own, which you sync back.

At this point, you have HA Postgres for both Prod and UAT ready before IIQ touches anything.
