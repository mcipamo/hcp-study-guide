# Architecture Overview

ROSA HCP is realized through **three distinct cluster tiers** that work together. Understanding this layered model is the foundation for everything else.

---

## The Three-Tier Model

```mermaid
graph TD
    OCM["🌐 OCM / Clusters Service\nCustomer-facing API\nRuns preflight checks\nOrchestrates provisioning"]

    SC["<b>Service Cluster</b>\nhs-sc-*\nONE per AWS region\nRuns ACM Hub\nManages all MCs in region"]

    MC["<b>Management Cluster</b>\nhs-mc-*\nMULTIPLE per region\nRuns control plane PODS\nUp to 80 HCPs per MC"]

    DP["<b>Data Plane</b>\nCustomer AWS Account\nWorker EC2 nodes\nCustomer workloads"]

    OCM -->|"Creates ManagedCluster + ManifestWork CRs"| SC
    SC -->|"ManifestWork propagates\nHostedCluster + NodePool CRs"| MC
    MC -->|"CAPI provisions\nEC2 instances via AWSMachine CRs"| DP
    MC <-->|"Konnectivity tunnel\n(PrivateLink)"| DP

    style OCM fill:#ede7f6,color:#4a148c,stroke:#7b1fa2
    style SC fill:#e3f2fd,color:#0d47a1,stroke:#1565c0
    style MC fill:#e8f5e9,color:#1b5e20,stroke:#2e7d32
    style DP fill:#fff3e0,color:#bf360c,stroke:#e65100
```

---

## Tier 1: OCM / Clusters Service

**Role**: Customer-facing API and orchestrator.

When a customer runs `rosa create cluster --hosted-cp`, OCM:

1. Validates the request (preflight checks against AWS)
2. Selects a Service Cluster in the customer's region (via OSD Fleet Manager provision shards)
3. Creates a `ManagedCluster` CR on the Service Cluster
4. Creates supporting secrets (`signing-key`, `root-CA`) on the Service Cluster
5. Watches the cluster status and reports back to the customer

!!! tip "First place to look for install/delete failures"
    CS (Clusters Service) logs on Grafana (Loki) are often the first indicator of where a provisioning or deprovisioning got stuck.

---

## Tier 2: Service Cluster (hs-sc-*)

**Role**: ACM hub that orchestrates Management Clusters in its region.

```mermaid
graph LR
    subgraph SC["Service Cluster (hs-sc-*)"]
        ACM["ACM Hub"]
        MC1["ManagedCluster CR\n(per hosted cluster)"]
        MW1["ManifestWork #1\nhostedCluster\n→ HostedCluster + NodePool CRs"]
        MW2["ManifestWork #2\nklusterlet\n→ Klusterlet pods + ACM Policies"]
    end

    ACM --> MC1
    MC1 --> MW1
    MC1 --> MW2
    MW1 -->|"Applied to MC"| MCT["Management Cluster"]
    MW2 -->|"Applied to MC"| MCT

    style SC fill:#e3f2fd,stroke:#1565c0
    style MCT fill:#e8f5e9,stroke:#2e7d32
```

Key properties:
- **One SC per AWS region** (e.g., `hs-sc-1sek0vq5g` in `us-east-1`)
- Manages multiple Management Clusters via `managedclusters.cluster.open-cluster-management.io`
- For each hosted cluster, creates **two ManifestWorks** in the namespace of the target MC:

| ManifestWork | Name Pattern | What it deploys |
|---|---|---|
| hostedCluster | `${clusterid}` | `HostedCluster` + `NodePool` CRs on the MC |
| klusterlet | `${clusterid}-hosted-klusterlet` | Klusterlet pods + ACM policies on the MC |

---

## Tier 3: Management Cluster (hs-mc-*)

**Role**: Runs the actual control plane pods for all hosted clusters assigned to it.

```mermaid
graph TD
    subgraph MC["Management Cluster (hs-mc-*)"]
        HO["HyperShift Operator\n(hypershift namespace)"]

        subgraph NS1["ocm-ENV-CLUSTERID namespace"]
            HC["HostedCluster CR"]
            NP["NodePool CR"]
            SEC["Secrets: SSH, admin-kubeconfig"]
        end

        subgraph NS2["ocm-ENV-CLUSTERID-NAME namespace"]
            CPO["control-plane-operator"]
            ETCD["etcd (3 pods)"]
            KAS["kube-apiserver (3 pods)"]
            KCM["kube-controller-manager"]
            KSCHED["kube-scheduler"]
            CAPI["capi-provider"]
            KONN["konnectivity-server/agent"]
            IGN["ignition-server"]
            HCCO["hosted-cluster-config-operator"]
        end

        subgraph NS3["klusterlet-CLUSTERID namespace"]
            KL["Klusterlet pods"]
            POL["ACM Policies"]
        end

        HO --> NS1
        HC --> NS2
    end

    style MC fill:#f1f8e9,stroke:#2e7d32
    style NS1 fill:#fff8e1,stroke:#f9a825
    style NS2 fill:#e8f5e9,stroke:#43a047
    style NS3 fill:#e3f2fd,stroke:#1976d2
```

Key properties:
- **Multiple MCs per region** — Fleet Manager adds more as demand grows
- Each MC can host **up to ~80 Hosted Control Planes**
- Every hosted cluster gets **exactly 3 namespaces** on the MC

### The Three Namespaces per Hosted Cluster

| # | Pattern | Contains |
|---|---------|----------|
| 1 | `ocm-${env}-${clusterid}` | `HostedCluster` CR, `NodePool` CR, SSH secret, admin-kubeconfig secret |
| 2 | `ocm-${env}-${clusterid}-${clustername}` | **All control plane pods** (etcd, kube-apiserver, etc.) |
| 3 | `klusterlet-${clusterid}` | Klusterlet pods, ACM compliance policies |

---

## Tier 4: Data Plane (Customer AWS)

**Role**: Where the customer's worker nodes and workloads run.

- EC2 instances provisioned by CAPI in the customer's AWS account
- Connected to the control plane via **PrivateLink** (see [Networking](networking.md))
- The customer never sees the Management or Service Cluster — only their worker nodes

---

## Fleet Manager and Capacity

**OSD Fleet Manager** orchestrates SC and MC lifecycle:

```mermaid
graph LR
    FM["OSD Fleet Manager"]
    SC1["SC us-east-1"]
    SC2["SC us-east-2"]
    MC1["MC 1\n(80 HCPs)"]
    MC2["MC 2\n(80 HCPs)"]
    MC3["MC 3\n(new)"]

    FM --> SC1
    FM --> SC2
    SC1 --> MC1
    SC1 --> MC2
    MC2 -->|"Saturation threshold\nhit → create new MC"| MC3

    style FM fill:#ede7f6,color:#4a148c
    style MC3 fill:#fff9c4,stroke:#f57f17
```

!!! note "MC Saturation Threshold"
    When an MC reaches its saturation threshold (~80 HCPs), Fleet Manager automatically provisions a new MC under the same Service Cluster. This is transparent to customers.

---

## How to Find These Clusters

```bash
# Find the Service Cluster for a region
ocm list cluster -p search="name like 'hs-sc%'" -p search="region.id='us-east-1'"

# Find the Management Cluster for a hosted cluster
ocm describe cluster $CLUSTER_NAME | grep 'Management Cluster'

# List all MCs managed by a given SC
ocm get /api/osd_fleet_mgmt/v1/management_clusters \
  -p search="parent.cluster_id='$SC_CLUSTER_ID'"

# List all hosted clusters on a given MC
ocm list clusters \
  -p "hypershift.management_cluster LIKE '${MC_NAME}'" \
  --columns id,name
```
