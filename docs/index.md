<div class="hero-banner">
  <h1>🔴 ROSA HCP Study Guide</h1>
  <p>Understanding how ROSA with Hosted Control Planes really works — from architecture to operations.</p>
</div>

# Welcome

This guide is designed for **Red Hat Managed Services engineers** who want to deeply understand how ROSA HCP (Hosted Control Planes) works — not just what commands to run, but *why* the system is built the way it is.

---

## Why HCP Exists

In a traditional ROSA (Classic) cluster, the **control plane runs inside the customer's AWS account** on three dedicated EC2 master nodes. This has several drawbacks:

- Customers pay for master nodes they never directly touch
- Every cluster needs ~45 minutes to provision (masters must boot and configure)
- SRE cannot easily access or repair a broken control plane
- Each cluster carries its own monitoring stack that customers can break
- Scaling to thousands of clusters requires thousands of master-node triples

HCP solves all of this by **moving the control plane out of the customer's account** and into Red Hat-managed clusters, while keeping worker nodes in the customer's account.

```mermaid
graph LR
    subgraph classic["Classic ROSA (before)"]
        direction TB
        M1[Master 1\nCustomer AWS]
        M2[Master 2\nCustomer AWS]
        M3[Master 3\nCustomer AWS]
        W1[Workers\nCustomer AWS]
        M1 --- M2 --- M3 --- W1
    end

    subgraph hcp["ROSA HCP (now)"]
        direction TB
        CP["Control Plane Pods\nRed Hat Managed MC"]
        W2["Workers\nCustomer AWS"]
        CP -->|PrivateLink| W2
    end

    style classic fill:#fff3e0,stroke:#e65100
    style hcp fill:#e8f5e9,stroke:#2e7d32
```

## The Mental Model

> Think of HCP like a **shared apartment building**.
> 
> - **Red Hat** owns and maintains the building foundation, roof, and utilities (control plane).
> - **Each customer** rents their own apartment (worker nodes) and does what they want inside it.
> - Customers never touch the foundation — they can't break what they can't access.

---

## Key Concepts at a Glance

<div class="concept-grid">
  <div class="concept-card">
    <h4>Service Cluster</h4>
    <p>One per AWS region. Runs ACM hub. Manages all Management Clusters in the region via ManifestWorks.</p>
  </div>
  <div class="concept-card">
    <h4>Management Cluster</h4>
    <p>Runs the actual control plane pods for up to 80 HCPs. Owned by Red Hat, never visible to customers.</p>
  </div>
  <div class="concept-card">
    <h4>HostedCluster CR</h4>
    <p>The primary Kubernetes resource representing a customer's cluster. Lives on the Management Cluster.</p>
  </div>
  <div class="concept-card">
    <h4>NodePool CR</h4>
    <p>Defines worker nodes: count, instance type, OCP version. Unique to HyperShift — no Classic equivalent.</p>
  </div>
  <div class="concept-card">
    <h4>Konnectivity</h4>
    <p>Bidirectional tunnel between the control plane (MC) and worker nodes (customer AWS). The bridge between worlds.</p>
  </div>
  <div class="concept-card">
    <h4>RHOBS</h4>
    <p>Centralized Red Hat Observability Service. All HCP metrics flow here — customers can't break it.</p>
  </div>
</div>

---

## How to Use This Guide

| I want to understand... | Go to |
|-------------------------|-------|
| The overall 3-cluster structure | [Architecture Overview](architecture.md) |
| What runs inside the control plane | [Control Plane Deep Dive](control-plane.md) |
| How PrivateLink and Konnectivity work | [Networking](networking.md) |
| How a cluster is created end-to-end | [Cluster Lifecycle](lifecycle.md) |
| How nodes get assigned and sized | [Node Scheduling & Sizing](scheduling.md) |
| How monitoring and alerting work | [Monitoring & Alerting](monitoring.md) |
| How HCP differs from Classic ROSA | [HCP vs Classic ROSA](vs-classic.md) |
| Quick operational commands | [Operations Reference](operations.md) |

---

!!! info "Source"
    This guide is based on the internal `ops-sop/hypershift/knowledge_base` SOPs.
