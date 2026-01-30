

## For Azure K8S (AKS) workloads vs Confluent Kafka

<img width="1082" height="442" alt="image" src="https://github.com/user-attachments/assets/30d5567b-cc40-44ff-bef2-3673d5fd6ecb" />

-----------

> **AKS + Strimzi + Managed Identity + Premium SSD**
> Production-grade P2P Kafka platform (3 AZ)

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

-----------

# Cost


**Kafka layer**

-   3 Kafka brokers (1 per AZ)
    
-   3 ZooKeeper nodes
    
-   Dedicated Kafka node pool
    
-   VM size: `Standard_D8s_v5`
    
-   Disk: 512 Gi Premium SSD per broker
    

**Platform**

-   AKS control plane (paid)
    
-   Prometheus + Grafana
    
-   MirrorMaker 2 (2 replicas)
    

No traffic egress to internet (private VNet).


## 1. AKS Control Plane

| Item                   | Cost             |
| ---------------------- | ---------------- |
| AKS cluster management | **~$73 / month** |

> Fixed cost per cluster. Doesn’t change with nodes.

## 2. Kafka Broker Node Pool (BIGGEST COST)

Compute (Kafka brokers)

| Item              | Qty | Unit cost | Monthly   |
| ----------------- | --- | --------- | --------- |
| `Standard_D8s_v5` | 3   | ~$280     | **~$840** |


-   8 vCPU / 32 GB RAM
    
-   One broker per node (best practice)

## Storage (Premium SSD)

| Disk               | Qty | Unit cost | Monthly   |
| ------------------ | --- | --------- | --------- |
| 512 Gi Premium SSD | 3   | ~$70      | **~$210** |

> Disk size determines IOPS → this is worth it for Kafka.

## 3. ZooKeeper Node Pool

You can reuse Kafka nodes or separate them.
(Production usually separates.)

### Compute

| Item              | Qty | Unit cost | Monthly   |
| ----------------- | --- | --------- | --------- |
| `Standard_D4s_v5` | 3   | ~$140     | **~$420** |

### Storage
| Disk               | Qty | Monthly  |
| ------------------ | --- | -------- |
| 128 Gi Premium SSD | 3   | **~$45** |


## 4. MirrorMaker 2 (DR Replication)

Runs as pods on non-Kafka node pool.

| Item             | Monthly      |
| ---------------- | ------------ |
| Compute (2 pods) | **~$80–120** |

Cost depends on where you schedule it.

## 5. Prometheus + Grafana

| Component             | Monthly |
| --------------------- | ------- |
| Prometheus (stateful) | ~$50    |
| Grafana               | ~$20    |

> Or Azure Managed Prometheus + Grafana (slightly higher, less ops).

## 6. Networking & Misc

| Item                      | Monthly |
| ------------------------- | ------- |
| Load balancers (internal) | ~$20    |
| Private DNS, VNet         | ~$10–20 |

## TOTAL MONTHLY COST (Realistic)

🔹 Baseline Production Kafka Platform

| Layer             | Cost |
| ----------------- | ---- |
| AKS control plane | $73  |
| Kafka brokers     | $840 |
| Kafka storage     | $210 |
| ZooKeeper compute | $420 |
| ZooKeeper storage | $45  |
| MM2               | $100 |
| Observability     | $70  |
| Networking        | $30  |

> Total: ~$1,800 – $1,900 / month


## Cost Optimization (Safe vs Unsafe)

### Safe optimizations

-   Reduce disk size if retention is low
    
-   Use `D4s_v5` for non-critical environments
    
-   Share ZK with Kafka nodes (small clusters)
    

### Unsafe (don’t do this for P2P)

-   Spot VMs
    
-   Autoscaling Kafka nodes
    
-   Standard SSD
    
-   Single AZ

## Compare With Alternatives

| Option                                       | Monthly Cost | Ops       |
| -------------------------------------------- | ------------ | --------- |
| **AKS + Strimzi (this)**                     | ~$1.9k       | Medium    |
| Managed Kafka (Azure Event Hubs / Confluent) | $2.5k–4k     | Low       |
| DIY VMs                                      | ~$1.5k       | High pain |

------------------------------------------

## Compare with Confluent Cloud pricing

| Dimension          | AKS + Strimzi | Confluent Cloud |
| ------------------ | ------------- | --------------- |
| Monthly cost       | **~$1.9k**    | **~$3k+**       |
| Throughput pricing | None          |  Metered        |
| Ops effort         | Medium        | Very low        |
| Kafka features     | Full OSS      | Full + extras   |
| Schema Registry    | DIY           | Built-in        |
| DR                 | MirrorMaker 2 | Cluster Linking |
| Lock-in            | None          | Medium          |
| Scaling traffic    | Cheap         | Expensive       |
| Audit / UI         | DIY           | Excellent       |

## Cost per 1M Payments

| Monthly Payments | AKS + Strimzi | Confluent Cloud |
| ---------------- | ------------- | --------------- |
| 10M              | $190          | **$86**         |
| 50M              | $38           | **$26**         |
| 100M             | $19           | **$16**         |
| 500M             | **$3.8**      | $11             |

## Break-Even Point (This Is the Key Insight)

-   **Below ~30–40M payments/month**  
       **Confluent Cloud is cheaper + easier**
    
-   **Above ~60–80M payments/month**  
      **AKS + Strimzi wins on cost**
    
-   **At scale (100M+)**  
     Strimzi is **2–3× cheaper per payment**
    
-----------------------------

## For AWS workloads

> **EKS + Strimzi + gp3 + IRSA**

