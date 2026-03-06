# 1. Node Prep / Cluster Baseline (Before Any "Tools")

You already have Ansible, so use it to normalize the nodes:

## Label and (optionally) taint nodes

Decide your zoning:

Nodes 1–3: env=prod (Prod app + Prod DB)  
Nodes 4–5: env=uat (UAT app + UAT DB)  
Node 6: env=mgmt (Argo CD, Ingress, monitoring, etc.)

Example:

```bash
kubectl label nodes node1 node2 node3 env=prod
kubectl label nodes node4 node5 env=uat
kubectl label nodes node6 env=mgmt
```

(You can add taints later if you want stricter separation.)

## Storage class(es)

Before CloudNativePG and IIQ, you need:

A reliable default StorageClass for general workloads.  
Ideally a fast StorageClass for Prod DB (NVMe/SSD with good IOPS).  
Optionally a separate class for UAT DB.

Make sure they are present and marked appropriately (isDefaultClass where needed).

## Cluster DNS / time / logging basics

Ensure NTP is set correctly from Ansible (time skew kills clusters).  
Confirm cluster DNS (CoreDNS) is healthy.  
Set baseline log shipping (even if just node logs → central syslog or similar).

Once this is stable, move to tools.
