# Cluster Lifecycle

This page walks through what actually happens when a ROSA HCP cluster is created and deleted — the sequence of components involved, the CRs that get created, and where to look when something goes wrong.

---

## Creation: End-to-End Sequence

```mermaid
sequenceDiagram
    participant C as Customer
    participant OCM as OCM / Clusters Service
    participant CS_Checks as CS Preflight Checks
    participant FM as OSD Fleet Manager
    participant SC as Service Cluster<br/>(hs-sc-*)
    participant ACM as ACM Hub (on SC)
    participant MC as Management Cluster<br/>(hs-mc-*)
    participant HO as HyperShift Operator<br/>(on MC)
    participant CPO as control-plane-operator<br/>(on MC)
    participant CAPI as capi-provider<br/>(on MC)
    participant DP as Customer AWS

    C->>OCM: rosa create cluster --hosted-cp
    OCM->>CS_Checks: Run 24 AWS preflight checks
    Note right of CS_Checks: VPC exists? Subnets tagged? OIDC configured?<br/>IAM roles present? DNS zone exists? Quota checks?
    CS_Checks-->>OCM: All checks pass

    OCM->>FM: Select provision shard (SC for region)
    FM-->>OCM: Returns SC endpoint

    OCM->>SC: Create ManagedCluster CR<br/>+ supporting secrets
    ACM->>MC: Create ManifestWork #1<br/>(HostedCluster + NodePool CRs)
    ACM->>MC: Create ManifestWork #2<br/>(Klusterlet CRs)

    MC->>HO: HyperShift Operator reconciles HostedCluster
    HO->>MC: Create HCP namespace<br/>+ hostedControlPlane namespace
    HO-->>MC: Create HostedControlPlane CR

    MC->>CPO: CPO reconciles HostedControlPlane CR
    CPO->>MC: Deploy etcd StatefulSet (3 pods)
    CPO->>MC: Deploy kube-apiserver (3 pods)
    CPO->>MC: Deploy kube-controller-manager, kube-scheduler
    CPO->>MC: Deploy ignition-server, konnectivity-server

    Note over CPO,MC: Control plane becomes Available ✓

    MC->>CAPI: CAPI provider reconciles NodePool
    CAPI->>DP: Provision EC2 instances via AWSMachine CRs

    DP->>MC: Workers boot, fetch ignition config from ignition-server
    DP->>MC: Workers connect via PrivateLink (kubelet → kube-apiserver)
    DP->>MC: Workers establish konnectivity tunnel

    MC->>OCM: HostedCluster.Status → Available: True
    OCM-->>C: Cluster Ready!
```

---

## The 24 Preflight Checks

Before any resource is created, Clusters Service validates the customer's AWS environment. If any check fails, the cluster creation is rejected with an actionable error.

```mermaid
graph LR
    subgraph Networking["Networking"]
        VPC["VPC exists"]
        SUBNETS["Subnets tagged\ncorrectly"]
        DNS["DNS zone\nregistered in Route53"]
    end

    subgraph IAM["IAM / Auth"]
        OIDC["OIDC config\ncreated"]
        ROLES["Required IAM\nroles present"]
        PERMS["Roles have\ncorrect policies"]
    end

    subgraph Quota["Quota / Capacity"]
        VPCQUOTA["VPC quota\nnot exceeded"]
        ECQUOTA["EC2 quota for\nrequested instance type"]
        EIPQUOTA["Elastic IP quota"]
    end

    subgraph KMS["Security"]
        KMS2["KMS key accessible\n(if etcd encryption enabled)"]
        SG["Security groups\nallow required traffic"]
    end
```

!!! tip "Where preflight failures appear"
    Preflight check failures are returned synchronously to the OCM API and surfaced in the `rosa create cluster` output. Look for `preflight check failed:` messages. In logs, check `clusters-service` on the Grafana Loki stack.

---

## What Gets Created Where

```mermaid
graph LR
    subgraph OCM["OCM / CS Database"]
        CDATA["Cluster record\n(account, config, status)"]
    end

    subgraph SC["Service Cluster Namespace\nocm-env-clusterid"]
        MANAGED["ManagedCluster CR"]
        MW1["ManifestWork: hostedCluster"]
        MW2["ManifestWork: hosted-klusterlet"]
        SECRETS_SC["Secrets: signing-key, root-ca"]
    end

    subgraph MC_NS1["MC: ocm-env-clusterid"]
        HC["HostedCluster CR"]
        NP["NodePool CR"]
        SSH["Secret: ssh-key"]
        KUBE["Secret: admin-kubeconfig"]
    end

    subgraph MC_NS2["MC: ocm-env-clusterid-name"]
        ETCD["StatefulSet: etcd (3 pods)"]
        KAS2["Deployment: kube-apiserver"]
        ALL_CP["...all control plane pods"]
        AWSEP["AWSEndpointService CR"]
    end

    subgraph MC_NS3["MC: klusterlet-clusterid"]
        KL["Klusterlet pods"]
        POLICIES["ACM Policies"]
    end

    subgraph Customer["Customer AWS Account"]
        EC2["EC2 worker instances"]
        NLB2["NLB (for PrivateLink)"]
        VPCE["VPC Endpoint"]
    end

    OCM --> SC
    SC --> MC_NS1
    MC_NS1 --> MC_NS2
    MC_NS2 --> Customer
```

---

## Deletion: End-to-End Sequence

Deletion is the reverse of creation, but the order matters to avoid orphaning resources.

```mermaid
sequenceDiagram
    participant C as Customer
    participant OCM as OCM / Clusters Service
    participant SC as Service Cluster
    participant MC as Management Cluster
    participant CAPI as capi-provider
    participant DP as Customer AWS

    C->>OCM: rosa delete cluster
    OCM->>SC: Delete ManagedCluster CR

    SC->>MC: Propagate deletion via ManifestWork
    MC->>CAPI: Trigger NodePool scale-down
    CAPI->>DP: Terminate EC2 worker instances

    Note over DP,MC: Workers drain and terminate

    MC->>MC: Delete HCP namespace pods (etcd, kube-apiserver, etc.)
    MC->>MC: Delete HCP namespace
    MC->>MC: Delete HC namespace

    SC->>SC: Delete ManifestWorks
    SC->>SC: Delete ManagedCluster CR

    OCM->>OCM: Update cluster status → Uninstalled
    OCM->>C: Cluster deleted
```

!!! warning "OIDC cleanup is separate"
    After cluster deletion, the OIDC configuration in AWS remains. It must be cleaned up manually:
    ```bash
    rosa delete oidc-config --oidc-config-id $OIDC_ID
    # or
    rosa delete oidc-provider -c $CLUSTER_ID
    ```

---

## Stuck States and Recovery

### Stuck in `Installing`

```mermaid
flowchart TD
    STUCK["Cluster stuck\nInstalling > 30 min"]
    CS["Check CS logs\n(Loki/Grafana)"]
    HC["Check HostedCluster\nconditions on MC"]
    CPO_LOG["Check CPO logs\nin HCP namespace"]
    CAPI_LOG["Check capi-provider logs\nfor EC2 provisioning errors"]
    ETCD_LOG["Check etcd pod logs\nfor leader election issues"]

    STUCK --> CS
    CS -->|"ManifestWork not created"| SC_ISSUE["SC issue:\ncheck ACM hub pods"]
    CS -->|"ManifestWork created"| HC
    HC -->|"HostedControlPlane\nnot Available"| CPO_LOG
    CPO_LOG -->|"etcd issue"| ETCD_LOG
    CPO_LOG -->|"CSR not approved"| MA["Check machine-approver\npod logs"]
    HC -->|"workers not joining"| CAPI_LOG
```

### Stuck `Uninstalling`

A common cause is a finalizer that isn't being cleared.

```bash
# Check what's blocking deletion
oc get hostedcluster -n $HC_NS -o yaml | grep -A 5 finalizers

# Check if NodePool resources are draining
oc get machines -n $HC_NS
oc get awsmachines -n $HC_NS

# Check capi-provider for errors terminating EC2
oc logs -n $HCP_NS deploy/capi-provider | tail -100
```

!!! danger "Manual finalizer removal — last resort"
    Only remove finalizers if CAPI has already confirmed the EC2 instances are terminated in the AWS console.
    ```bash
    oc patch hostedcluster -n $HC_NS $CLUSTER_NAME \
      --type=json -p '[{"op":"remove","path":"/metadata/finalizers"}]'
    ```

---

## Cluster Upgrade Lifecycle

Upgrades in HCP are decoupled: **control plane and workers upgrade independently**.

```mermaid
sequenceDiagram
    participant C as Customer
    participant OCM as OCM
    participant HC as HostedCluster CR
    participant CVO as cluster-version-operator
    participant NP as NodePool CR
    participant CAPI as capi-provider

    C->>OCM: rosa upgrade cluster --version 4.16.5
    OCM->>HC: Update HostedCluster.spec.release.image

    HC->>CVO: CVO detects new version
    CVO->>CVO: Apply new OCP manifests
    Note over CVO: Control plane upgrades first<br/>(kube-apiserver, etcd, operators)
    CVO-->>HC: Status: ControlPlaneUpgradeComplete

    OCM->>NP: Update NodePool.spec.release.image
    NP->>CAPI: Rolling replace worker nodes
    CAPI->>CAPI: Provision new nodes at new version
    CAPI->>CAPI: Drain and terminate old nodes
    CAPI-->>NP: All workers at new version

    NP-->>OCM: Upgrade complete
    OCM-->>C: Cluster upgraded to 4.16.5
```

Key difference from Classic: the control plane upgrade does **not** require worker node drains — workers continue serving traffic throughout the CP upgrade.
