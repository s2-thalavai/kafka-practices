

## For Azure K8S (AKS) workloads

> **AKS + Strimzi + Managed Identity + Premium SSD**

**Key principles**

-   One Kafka broker per node
    
-   Nodes spread across AZs (1/2/3)
    
-   Premium SSD for predictable latency
    
-   Managed Identity for _cloud_ access
    
-   TLS certs for _Kafka_ access



## 1. AKS Cluster Creation (Non-Negotiable Settings)

### Required flags

-   **Availability Zones enabled**
    
-   **Azure CNI (not kubenet)**
    
-   **Standard Load Balancer**
    
-   **OIDC + Workload Identity enabled**
    

Example (conceptual):

```
AZs:  1,2,3  
Network:  Azure  CNI  
LB SKU:  Standard
``` 

> If AZs are not enabled → **Kafka rack awareness is useless**

----------

## 2. Dedicated Kafka Node Pool (Most Important AKS Step)

### Node Pool: `kafka`

| Setting     | Value                          |
| ----------- | ------------------------------ |
| VM size     | `Standard_D8s_v5` (or D16s_v5) |
| Zones       | `1,2,3`                        |
| OS disk     | Premium SSD                    |
| Autoscaling | Disabled                     |
| Taints      | `kafka=true:NoSchedule`        |



### Why

-   Kafka hates noisy neighbors
    
-   Predictable CPU & disk
    
-   No scale-down surprises
    

### Taint

```yaml
spec:
  tolerations:
    - key: "kafka"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
``` 

(Strimzi already tolerates taints correctly.)


## 3. StorageClass: Premium SSD (AKS Way)

### Use built-in class

```
managed-premium
``` 

### In your Helm `values.yaml`

```yaml
kafka:
  storage:
    class: managed-premium
    size: 512Gi
``` 

### Why Premium SSD

-   Low latency
    
-   Predictable IOPS
    
-   Proven with Kafka on AKS
    

> Ultra Disk is **overkill** unless you’re pushing insane throughput.



## 4. Rack Awareness (Already Correct – Just Verify)

You already have this ():

```yaml
rack:
  topologyKey: topology.kubernetes.io/zone
``` 

### Validate nodes

```bash
kubectl get nodes -L topology.kubernetes.io/zone
``` 

Expected:

```
node-1   zone=1
node-2   zone=2
node-3   zone=3
``` 

----------

## 5. Managed Identity (What It Is / What It Is NOT)

### Managed Identity = for Azure services

Used by:

-   Prometheus remote write
    
-   Blob Storage sinks
    
-   Backup jobs
    
-   Azure Monitor
    

### NOT used for:

-  Kafka authentication  
 - Topic ACLs  
 - Consumer groups

Kafka auth stays **TLS cert-based** (correct and cloud-agnostic).

----------

## Workload Identity Setup (Recommended)

-   Enable AKS OIDC
    
-   Create **User Assigned Managed Identity**
    
-   Bind via ServiceAccount
    

Example:

```yaml
serviceAccount:
  annotations:
    azure.workload.identity/client-id: <client-id>
``` 

Use this for:

-   Kafka Connect sinks to Blob
    
-   MM2 metrics export
    
-   Backup / export jobs



## 6. Networking: Keep Kafka Private

### Internal traffic

-   `ClusterIP` only
    
-   No public exposure
    

### Cross-service access

-   Same VNet
    
-   Or VNet peering
    
-   Private DNS
    

>  **Do NOT expose Kafka publicly for payments**

----------

## 7. Prometheus + Grafana on AKS

## Recommended

-   `kube-prometheus-stack`
    
-   Storage on `managed-premium`
    
-   Node selector: non-Kafka nodes
    

``` yaml
nodeSelector:
  workload: monitoring
```

### Alerts that matter most (AKS)

-   ISR shrink
    
-   Disk usage
    
-   Broker down
    
-   MM2 lag
    

AKS behaves well here—no cloud-specific quirks.


## 8. MirrorMaker 2 on AKS (DR)

###  AKS → AKS (Cross-Region)

| Layer        | Choice            |
| ------------ | ----------------- |
| Connectivity | VNet Peering      |
| DNS          | Private DNS Zones |
| Security     | TLS               |
| Auth         | Kafka certs       |


### Latency note

-   MM2 tolerates latency
    
-   Payments **must not** depend on DR path


## 9. Resource Sizing (AKS-Tested)

## Kafka Broker Pod

```
cpu:  4–8  memory:  16–32Gi
``` 

## Zookeeper / KRaft

```
cpu:  1–2  memory:  4–8Gi
``` 

## MM2

```
cpu:  1–2  memory:  4–8Gi
``` 

One broker per node → **gold standard**


## AKS-Specific Gotchas

### Autoscaler scale-down

Disable for Kafka node pool.

### Disk resizing in-place

Kafka doesn’t love it — plan capacity upfront.

### Mixing workloads

Never run apps + Kafka on same pool.

### Zone outage

Your rack-aware Kafka survives a full AZ loss.

----------

## Final AKS Production Checklist

✔ AKS with AZs enabled  
✔ Dedicated Kafka node pool  
✔ Premium SSD (`managed-premium`)  
✔ Rack awareness active  
✔ TLS Kafka auth  
✔ Managed Identity for Azure access  
✔ Helm + ArgoCD GitOps  
✔ Prometheus + Grafana alerts


------------------------------------------


## For AWS workloads

> **EKS + Strimzi + gp3 + IRSA**

