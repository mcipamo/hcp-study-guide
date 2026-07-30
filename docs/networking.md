# Networking: PrivateLink & Konnectivity

The fundamental networking challenge in HCP is this: **the control plane (etcd, kube-apiserver) runs on a Red Hat-owned Management Cluster, while the worker nodes run in the customer's AWS account**. These two environments live in completely separate VPCs with no direct peering. This page explains how they communicate.

---

## The Two-Way Connectivity Problem

```mermaid
graph LR
    subgraph RH["Red Hat Managed Infrastructure"]
        MC["Management Cluster\n(Red Hat VPC)"]
        KAS["kube-apiserver\npod"]
        KSERVER["konnectivity-server\npod"]
    end

    subgraph Customer["Customer AWS Account"]
        W1["Worker Node 1"]
        W2["Worker Node 2"]
        W3["Worker Node 3"]
        KAGENT["konnectivity-agent\n(DaemonSet on workers)"]
    end

    KSERVER <-->|"gRPC over\nAWS PrivateLink"| KAGENT
    KAGENT <-->|"tunnels traffic"| W1
    KAGENT <-->|"tunnels traffic"| W2
    KAGENT <-->|"tunnels traffic"| W3
    KAS -->|"sends commands via tunnel"| KAGENT

    style RH fill:#e3f2fd,stroke:#1565c0
    style Customer fill:#fff3e0,stroke:#e65100
```

The solution uses **two complementary technologies**:

| Technology | Direction | Purpose |
|------------|-----------|---------|
| AWS PrivateLink | Customer → Red Hat | Exposes the MC's API endpoint privately into the customer's VPC |
| Konnectivity | Red Hat → Customer | Provides the reverse tunnel so the API server can reach pods on workers |

---

## AWS PrivateLink: Customer VPC → MC

**PrivateLink** lets the customer's workers reach the kube-apiserver without exposing it on the public internet.

```mermaid
sequenceDiagram
    participant W as Worker Node
    participant EP as VPC Endpoint<br/>(Customer Account)
    participant NLB as NLB<br/>(Red Hat Account)
    participant KAS as kube-apiserver<br/>(MC)

    W->>EP: kubelet → api.${cluster}.openshiftapps.com
    EP->>NLB: PrivateLink tunnel
    NLB->>KAS: Forward to kube-apiserver
    KAS-->>NLB: Response
    NLB-->>EP: PrivateLink return
    EP-->>W: Response to kubelet
```

Key points:
- Each hosted cluster gets its own **NLB** (Network Load Balancer) in the Red Hat VPC
- A **VPC Endpoint Service** is created from that NLB
- A **VPC Endpoint** is created in the customer's VPC pointing to the endpoint service
- Workers resolve `api.${cluster}.openshiftapps.com` → private VPC endpoint IP
- **No traffic crosses the public internet**

The CR that tracks this on the MC:
```bash
oc get awsendpointservice -n $CP_NS
oc describe awsendpointservice $CLUSTER_NAME -n $CP_NS
```

---

## Konnectivity: MC → Worker Nodes

The kube-apiserver needs to reach pods on worker nodes (e.g., for `oc exec`, `oc logs`, admission webhooks). Without Konnectivity, this would require a separate VPN or peering back the other way.

**Konnectivity** solves this by having workers establish a persistent outbound tunnel to the MC:

```mermaid
sequenceDiagram
    participant KA as konnectivity-agent<br/>(runs on workers)
    participant PL as PrivateLink<br/>Endpoint
    participant KS as konnectivity-server<br/>(runs in HCP namespace)
    participant KAS as kube-apiserver

    Note over KA,KS: Startup: agent establishes tunnel
    KA->>PL: Outbound gRPC connect
    PL->>KS: PrivateLink to konnectivity-server
    KS-->>KA: Tunnel established (persistent)

    Note over KAS,KA: Runtime: API server reaches a pod
    KAS->>KS: Send command to pod on worker
    KS->>KA: Route through tunnel
    KA->>KA: Deliver to target pod
    KA-->>KS: Response
    KS-->>KAS: Return to API server
```

Key points:
- `konnectivity-agent` runs as a **DaemonSet** on all worker nodes
- The agent dials out to `konnectivity-server` (in the HCP namespace on the MC) over PrivateLink
- The connection is **initiated by the worker** — no inbound firewall rules needed in the customer VPC
- **All traffic from API server → pod** is multiplexed through this single tunnel

---

## Zero Egress: No NAT, No IGW

For clusters in **Zero Egress** mode, worker nodes have **no internet access at all** — no NAT gateway, no Internet Gateway.

This poses a challenge: workers need to pull container images. The solution is mirroring:

```mermaid
graph LR
    Q["quay.io\n(Red Hat registry)"]
    CB["AWS CodeBuild Jobs\nocp-mirror-2 through ocp-mirror-8\nmirror_operators, mirror_candidate"]
    ECR["AWS ECR\n(customer account)"]
    W["Worker Nodes\n(no internet)"]
    CRED["ECR Credential Provider\n(manages auth tokens)"]

    Q -->|"mirror on release"| CB
    CB -->|"sync images"| ECR
    ECR -->|"private pull"| W
    CRED -->|"refreshes tokens"| ECR

    style Q fill:#ffebee,color:#b71c1c
    style ECR fill:#fff8e1,color:#e65100
    style W fill:#e3f2fd,color:#0d47a1
```

- Red Hat-owned CodeBuild jobs copy OCP release images from quay.io → customer ECR repositories
- Workers pull images from ECR over internal AWS networking (no internet)
- An ECR Credential Provider refreshes the `~/.docker/config.json` tokens periodically
- If a worker fails with `ImagePullBackOff`, check if the ECR mirror job ran for that release version

---

## Network Flows Summary

```mermaid
graph TD
    subgraph External["External / Public"]
        OCMAPI["OCM API\napi.openshift.com"]
        RHOBS["RHOBS\nRed Hat Observability"]
    end

    subgraph MC["Management Cluster (Red Hat)"]
        KAS["kube-apiserver"]
        KSERVER["konnectivity-server"]
        METRICS["metrics-forwarder\nproxy"]
    end

    subgraph Customer["Customer VPC"]
        VPCE["VPC Endpoint\n(PrivateLink)"]
        W["Workers"]
        KAGENT["konnectivity-agent"]
        MF["metrics-forwarder\n(DaemonSet)"]
    end

    W -->|"kubelet"| VPCE
    VPCE -->|"PrivateLink"| KAS
    KAGENT -->|"outbound tunnel"| KSERVER
    KAS -->|"exec/logs/webhooks"| KSERVER
    KSERVER -->|"via tunnel"| KAGENT
    KAGENT -->|"to pod"| W

    MF -->|"push metrics"| METRICS
    METRICS -->|"forward"| RHOBS

    style MC fill:#e8f5e9,stroke:#2e7d32
    style Customer fill:#fff3e0,stroke:#e65100
    style External fill:#f3e5f5,stroke:#7b1fa2
```

| Flow | Protocol | Path |
|------|----------|------|
| Worker kubelet → API | HTTPS | PrivateLink → MC NLB → kube-apiserver |
| API server → Pod (exec/logs) | gRPC | konnectivity tunnel |
| Worker → OCP images | HTTPS | ECR (internal, no internet) |
| Metrics → RHOBS | HTTPS | metrics-forwarder → PrivateLink → RHOBS |
| OCM → Provision cluster | HTTPS | OCM API → SC → MC (ManifestWork) |

---

## Troubleshooting Network Issues

```bash
# Check the PrivateLink endpoint service
oc get awsendpointservice -n $CP_NS

# Check konnectivity tunnel status
oc logs -n $CP_NS deploy/konnectivity-server
oc logs -n openshift-konnectivity-agent konnectivity-agent-$NODE

# Check if workers can reach the API server
oc exec -n $CP_NS -it debug-pod -- curl -k https://api.${CLUSTER}.openshiftapps.com:6443/healthz

# Check ECR credential provider (zero egress)
oc logs -n openshift-cloud-credential-operator -l app=cloud-credential-operator
```
