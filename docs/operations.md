# Operations Reference

Quick-reference commands for day-to-day HCP operations. This page assumes you have `oc` authenticated to the Management Cluster.

---

## Environment Variables

Set these once per session:

```bash
# The environment: staging, production
export ENV=production

# The cluster ID (from OCM)
export CLUSTER_ID=abc123def456

# The cluster name (from OCM)
export CLUSTER_NAME=my-hcp-cluster

# Derived namespace variables
export HC_NS="ocm-${ENV}-${CLUSTER_ID}"
export HCP_NS="ocm-${ENV}-${CLUSTER_ID}-${CLUSTER_NAME}"

# The Management Cluster (from OCM describe)
export MC_NAME=$(ocm describe cluster $CLUSTER_ID | grep "Management Cluster" | awk '{print $NF}')
```

---

## Cluster Discovery

```bash
# Find Service Cluster for a region
ocm list clusters -p "search=name like 'hs-sc-%' and region.id='us-east-1'"

# Find Management Cluster for a specific hosted cluster
ocm describe cluster $CLUSTER_ID | grep -i "management cluster"

# List all hosted clusters on a given MC
ocm list clusters \
  -p "hypershift.management_cluster LIKE '${MC_NAME}'" \
  --columns id,name,state,version

# List all MCs managed by an SC
ocm get /api/osd_fleet_mgmt/v1/management_clusters \
  -p search="parent.cluster_id='$SC_CLUSTER_ID'"
```

---

## HostedCluster Status

```bash
# Quick health check
oc get hostedcluster -n $HC_NS $CLUSTER_NAME

# Detailed conditions
oc get hostedcluster -n $HC_NS $CLUSTER_NAME \
  -o jsonpath='{.status.conditions}' | jq '.'

# Check availability
oc get hostedcluster -n $HC_NS $CLUSTER_NAME \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'

# Check all NodePools
oc get nodepool -n $HC_NS

# Check NodePool details
oc get nodepool -n $HC_NS -o wide
```

---

## Control Plane Health

```bash
# List all pods in the HCP namespace
oc get pods -n $HCP_NS

# Check for any non-Running pods
oc get pods -n $HCP_NS | grep -v Running | grep -v Completed

# Control plane operator (the reconciler — start here for most issues)
oc logs -n $HCP_NS deploy/control-plane-operator -f

# kube-apiserver
oc logs -n $HCP_NS -l app=kube-apiserver

# etcd health
oc exec -n $HCP_NS etcd-0 -- \
  etcdctl endpoint health \
  --cacert=/etc/etcd/tls/etcd-ca/ca.crt \
  --cert=/etc/etcd/tls/client/etcd-client.crt \
  --key=/etc/etcd/tls/client/etcd-client.key \
  --endpoints=https://etcd-client:2379

# etcd leader status
oc exec -n $HCP_NS etcd-0 -- \
  etcdctl endpoint status \
  --cacert=/etc/etcd/tls/etcd-ca/ca.crt \
  --cert=/etc/etcd/tls/client/etcd-client.crt \
  --key=/etc/etcd/tls/client/etcd-client.key \
  --endpoints=https://etcd-client:2379 -w table

# capi-provider (for worker provisioning issues)
oc logs -n $HCP_NS deploy/capi-provider
```

---

## Worker Nodes

```bash
# List all machines (worker CRs on MC, not in the hosted cluster)
oc get machines -n $HCP_NS

# List AWSMachines
oc get awsmachines -n $HCP_NS

# Check machine status
oc describe machine -n $HCP_NS $MACHINE_NAME

# Pending CSRs (new node joining)
# Run this against the HOSTED cluster, not the MC
oc get csr --kubeconfig $KUBECONFIG_HOSTED | grep Pending

# Approve a CSR manually (if machine-approver is stuck)
oc adm certificate approve $CSR_NAME --kubeconfig $KUBECONFIG_HOSTED
```

---

## Break-Glass Access

### Normal: Admin Kubeconfig

```bash
# Extract admin kubeconfig
oc get secret -n $HC_NS ${CLUSTER_NAME}-admin-kubeconfig \
  -o jsonpath='{.data.kubeconfig}' | base64 -d > /tmp/hcp-kubeconfig

# Use it
oc --kubeconfig /tmp/hcp-kubeconfig get nodes
```

### Emergency: SSM + crictl (when API is down)

```bash
# Find an EC2 instance ID for a worker node
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=*${CLUSTER_NAME}*" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]' \
  --output table

# Start SSM session
aws ssm start-session --target $INSTANCE_ID

# On the node: check kubelet
sudo systemctl status kubelet

# On the node: check containers
sudo crictl ps

# On the node: check logs
sudo journalctl -u kubelet -f
```

### OCM-Down: apply-bootstrap container

```bash
# If OCM is unavailable but the MC is accessible
oc exec -n $HCP_NS $KAS_POD -c apply-bootstrap -- \
  oc --kubeconfig /etc/kubernetes/static-pod-resources/kube-apiserver-certs/admin-kubeconfig get nodes
```

---

## Networking

```bash
# Check PrivateLink endpoint service status
oc get awsendpointservice -n $HCP_NS
oc describe awsendpointservice -n $HCP_NS

# Check konnectivity tunnel
oc logs -n $HCP_NS deploy/konnectivity-server | tail -50

# Check VPC endpoint in customer account
aws ec2 describe-vpc-endpoints \
  --filters "Name=tag:Name,Values=*${CLUSTER_NAME}*"
```

---

## Alerts and Monitoring

```bash
# Check if cluster alerts are suppressed
oc get hostedcluster -n $HC_NS $CLUSTER_NAME \
  -o jsonpath='{.metadata.annotations.hypershift\.openshift\.io/disable-cluster-alerts}'

# Suppress alerts (during maintenance)
oc annotate hostedcluster -n $HC_NS $CLUSTER_NAME \
  hypershift.openshift.io/disable-cluster-alerts=true

# Remove alert suppression
oc annotate hostedcluster -n $HC_NS $CLUSTER_NAME \
  hypershift.openshift.io/disable-cluster-alerts-

# Check MonitoringStack health
oc get monitoringstack -n $HCP_NS
```

---

## Maintenance Mode (SC/MC)

!!! danger "Maintenance mode affects ALL clusters on the MC"
    Placing an MC in maintenance mode affects every HCP on that MC.

```bash
# Check current maintenance state
ocm get /api/osd_fleet_mgmt/v1/management_clusters/$MC_ID

# Enable maintenance (via Fleet Manager API, SRE use only)
# This pauses HyperShift Operator reconciliation

# Check which clusters are affected
ocm list clusters \
  -p "hypershift.management_cluster LIKE '$MC_NAME'" \
  --columns id,name,state
```

---

## Cluster Lifecycle Commands

```bash
# Create a cluster (staging)
rosa create cluster \
  --hosted-cp \
  --cluster-name $CLUSTER_NAME \
  --sts \
  --mode auto \
  --region us-east-2 \
  --subnet-ids $SUBNET_IDS \
  --oidc-config-id $OIDC_ID

# Delete a cluster
rosa delete cluster -c $CLUSTER_NAME --yes

# Clean up OIDC (after deletion)
rosa delete oidc-config --oidc-config-id $OIDC_ID

# Upgrade control plane
rosa upgrade cluster -c $CLUSTER_NAME --version 4.16.5

# Upgrade workers (NodePool)
rosa upgrade machinepool -c $CLUSTER_NAME \
  --machinepool workers \
  --version 4.16.5
```

---

## Useful One-Liners

```bash
# Count HCPs per MC in a region
ocm list clusters \
  -p "region.id='us-east-1' and name like 'hs-mc-%'" \
  --columns name,id | while read MC_NAME MC_ID; do
  COUNT=$(ocm list clusters \
    -p "hypershift.management_cluster LIKE '$MC_NAME'" \
    --count-only 2>/dev/null)
  echo "$MC_NAME: $COUNT HCPs"
done

# Find all degraded HCPs in a region
ocm list clusters \
  -p "region.id='us-east-1' and state='Error'" \
  --columns id,name,state,version

# Get the admin kubeconfig for a cluster and export it
oc get secret -n $HC_NS ${CLUSTER_NAME}-admin-kubeconfig \
  -o jsonpath='{.data.kubeconfig}' \
  | base64 -d \
  | KUBECONFIG=/dev/stdin oc get nodes

# Watch all pods in HCP namespace coming up
watch -n 5 "oc get pods -n $HCP_NS | sort -k3"
```
