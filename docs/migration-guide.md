# Migrating to the cert-manager ACM Add-on

This guide covers migrating managed clusters that already have cert-manager
installed (manually, via Policy, or via Helm) to the native ACM add-on.

The add-on deploys `openshift-cert-manager-operator` via OLM Subscription in
the `open-cluster-management-agent-addon` namespace. Existing Certificate
resources are preserved throughout the migration because CRDs are not removed.

## 1. Pre-Migration Assessment

Run these commands **on each managed cluster** (or via hub with `--context`):

```bash
# Check for existing cert-manager Subscriptions
oc get subscription -A | grep cert-manager

# Check for installed CSVs
oc get csv -A | grep cert-manager

# Check which namespace the operator runs in
oc get pods -A | grep cert-manager

# Check for existing Certificates (these will be preserved)
oc get certificates -A
oc get issuers -A
oc get clusterissuers
```

Record the existing Subscription namespace and CSV name for each cluster.
The add-on will install into `open-cluster-management-agent-addon`, so any
Subscription in a different namespace must be removed first to avoid a
duplicate operator.

## 2. Migration Path A: Manual / Direct Installation

Use this path when cert-manager was installed directly via `oc apply`,
OperatorHub UI, or Helm chart (no ACM Policy managing it).

### Step 1: Remove the existing Subscription (per managed cluster)

```bash
CLUSTER_CTX="<managed-cluster-context>"
EXISTING_NS="<namespace-where-subscription-exists>"  # e.g., openshift-operators

# Delete the Subscription (stops OLM from managing upgrades)
oc --context=$CLUSTER_CTX delete subscription -n $EXISTING_NS \
  openshift-cert-manager-operator

# Delete the CSV (removes the operator Deployment, but keeps CRDs)
CSV_NAME=$(oc --context=$CLUSTER_CTX get csv -n $EXISTING_NS \
  -o name | grep cert-manager)
oc --context=$CLUSTER_CTX delete $CSV_NAME -n $EXISTING_NS
```

### Step 2: Verify CRDs and Certificates remain

```bash
# CRDs should still exist
oc --context=$CLUSTER_CTX get crd certificates.cert-manager.io

# Existing Certificates should still show their last-known status
oc --context=$CLUSTER_CTX get certificates -A
```

### Step 3: Enable the add-on for this cluster (on hub)

```bash
# Label the cluster for add-on placement
oc label managedcluster <cluster-name> cert-manager=enabled

# Verify the ManagedClusterAddOn is created
oc get managedclusteraddon -n <cluster-name> cert-manager-addon
```

### Step 4: Wait for the add-on operator to reconcile

The new operator installation picks up existing CRDs and Certificate
resources. No re-issuance occurs unless certificates are near expiry.

## 3. Migration Path B: Policy-Based Installation

Use this path when cert-manager is deployed via an ACM Policy that creates
and enforces a Subscription on managed clusters.

### Step 1: Disable the Policy (on hub)

```bash
# Option A: Set the Policy to inform-only (stops enforcement)
oc patch policy <cert-manager-policy-name> -n <policy-namespace> \
  --type merge -p '{"spec":{"remediationAction":"inform"}}'

# Option B: Delete the Policy entirely
oc delete policy <cert-manager-policy-name> -n <policy-namespace>
```

If using a PolicySet or PlacementBinding, remove those as well:

```bash
oc delete placementbinding <binding-name> -n <policy-namespace>
```

### Step 2: Remove existing Subscriptions on managed clusters

After the Policy is disabled, remove the Subscription and CSV from each
managed cluster (same commands as Path A, Step 1).

ManagedCluster resource names do not always match kubeconfig context names.
Build an explicit mapping before running the cleanup loop:

```bash
# Build a cluster-name → kubeconfig-context mapping.
# Replace the context values with the actual context names from your kubeconfig.
declare -A CLUSTER_CTX_MAP=(
  ["cluster1"]="cluster1-context"
  ["cluster2"]="cluster2-context"
  # Add one entry per managed cluster
)

EXISTING_NS="openshift-operators"  # adjust if your Policy used a different namespace

for cluster in "${!CLUSTER_CTX_MAP[@]}"; do
  ctx="${CLUSTER_CTX_MAP[$cluster]}"
  echo "=== $cluster (context: $ctx) ==="
  oc --context=$ctx delete subscription -n $EXISTING_NS \
    openshift-cert-manager-operator --ignore-not-found
  CSV=$(oc --context=$ctx get csv -n $EXISTING_NS -o name 2>/dev/null | grep cert-manager)
  [ -n "$CSV" ] && oc --context=$ctx delete $CSV -n $EXISTING_NS
done
```

### Step 3: Enable the add-on (on hub)

```bash
# Label clusters and deploy the add-on manifests
oc label managedcluster <cluster-name> cert-manager=enabled

# Or label all clusters at once
oc get managedcluster -o name | xargs -I{} oc label {} cert-manager=enabled
```

## 4. Post-Migration Verification

Run after the add-on reports AVAILABLE=True.

```bash
CLUSTER_CTX="<managed-cluster-context>"

# Verify no duplicate Subscriptions
oc --context=$CLUSTER_CTX get subscription -A | grep cert-manager
# Should show exactly ONE Subscription in open-cluster-management-agent-addon

# Verify CSV is Succeeded
oc --context=$CLUSTER_CTX get csv -n open-cluster-management-agent-addon \
  | grep cert-manager
# Phase should be "Succeeded"

# Verify operator pods are running
oc --context=$CLUSTER_CTX get pods -A | grep cert-manager

# Verify existing Certificates are still valid
oc --context=$CLUSTER_CTX get certificates -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status'
# All should show READY=True

# Verify Issuers are intact
oc --context=$CLUSTER_CTX get issuers -A
oc --context=$CLUSTER_CTX get clusterissuers
```

On the hub, verify via ManifestWork feedback:

```bash
oc get manifestwork -n <cluster-name> addon-cert-manager-addon-deploy-0 \
  -o json | jq '.status.resourceStatus.manifests[]
    | select(.resourceMeta.name=="openshift-cert-manager-operator")
    | .statusFeedback.values'
# state should be "AtLatestKnown"
```

## 5. Rollback

If the migration fails, revert to the previous installation method.

### Step 1: Remove the add-on (on hub)

```bash
# Remove the cluster label (prevents re-deployment)
oc label managedcluster <cluster-name> cert-manager-

# Or delete the ClusterManagementAddOn entirely (removes from all clusters)
oc delete clustermanagementaddon cert-manager-addon
```

### Step 2: Wait for cleanup

```bash
# Verify ManagedClusterAddOn is removed
oc get managedclusteraddon -n <cluster-name> cert-manager-addon
# Should return "not found" after 1-2 minutes
```

### Step 3: Re-apply the original installation

```bash
CLUSTER_CTX="<managed-cluster-context>"
ORIGINAL_NS="openshift-operators"  # use your original namespace

# Re-create the Subscription
oc --context=$CLUSTER_CTX apply -f <your-original-subscription.yaml>

# Or re-enable the Policy (if using Path B)
oc patch policy <cert-manager-policy-name> -n <policy-namespace> \
  --type merge -p '{"spec":{"remediationAction":"enforce"}}'
```

Existing CRDs and Certificate resources are not affected by this rollback.
The restored operator will resume reconciling them.
