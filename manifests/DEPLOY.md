# Quick Deployment Guide

This guide provides copy-paste commands to deploy the cert-manager AddOn to your ACM environment.

## Prerequisites Check

```bash
# Verify you're connected to the hub cluster
oc whoami --show-server

# Verify managed clusters are available
oc get managedcluster

# Expected output: All clusters show AVAILABLE: True
```

## Deployment Steps

### 1. Create Namespace

```bash
oc apply -f 01-namespace.yaml
```

**Verify:**
```bash
oc get namespace open-cluster-management-global-set
```

### 2. Create Placement

```bash
# Label target clusters first
oc label managedcluster <cluster-name> cert-manager=enabled

oc apply -f 03-placement.yaml

# Wait for placement to process
sleep 5

# Verify clusters are selected
oc get placement cert-manager-placement -n open-cluster-management-global-set

# Check placement decisions
oc get placementdecision -n open-cluster-management-global-set \
  -l cluster.open-cluster-management.io/placement=cert-manager-placement \
  -o jsonpath='{range .items[*].status.decisions[*]}{.clusterName}{"\n"}{end}'
```

**Expected output:** Names of your managed clusters labeled with `cert-manager=enabled`

### 3. Deploy AddOnTemplate

```bash
oc apply -f 04-addontemplate.yaml

# Verify
oc get addontemplate cert-manager-addon-template
```

### 4. Deploy ClusterManagementAddOn (Triggers Rollout)

```bash
oc apply -f 05-clustermanagementaddon.yaml

# Verify
oc get clustermanagementaddon cert-manager-addon -o yaml
```

**Check for:**
- `spec.installStrategy.type: Placements` (NOT Manual)
- `spec.installStrategy.placements` section exists

### 5. Monitor Rollout

```bash
# Watch ManagedClusterAddOn creation (refresh every 5 seconds)
watch -n 5 'oc get managedclusteraddon -A | grep cert-manager-addon'
```

Wait until all clusters show `AVAILABLE: True`. This indicates the ManifestWork
has been applied, but does **not** confirm the operator is fully installed —
OLM resolves the InstallPlan and CSV asynchronously after the Subscription is
created. Proceed to the next step to verify OLM installation.

### 6. Verify OLM Subscription and CSV

The Subscription status is surfaced via ManifestWork feedback. Check the
Subscription `state` and `installedCSV` fields:

```bash
# Check ManifestWork feedback for Subscription status
for cluster in $(oc get managedclusteraddon -A | grep cert-manager-addon | awk '{print $1}'); do
  echo "=== $cluster ==="
  oc get manifestwork -n $cluster addon-cert-manager-addon-deploy-0 \
    -o json | jq '.status.resourceStatus.manifests[]
      | select(.resourceMeta.name=="openshift-cert-manager-operator")
      | .statusFeedback.values' 2>/dev/null || echo "  (no feedback yet)"
done
```

**Expected output per cluster:**
- `state`: `AtLatestKnown` — OLM has resolved the Subscription
- `installedCSV`: e.g. `cert-manager-operator.v1.x.y` — the operator CSV is installed

If feedback is not yet available, verify directly on the managed cluster:

```bash
# Switch context to managed cluster
oc config use-context <cluster-context>

# Check Subscription state
oc get subscription -n open-cluster-management-agent-addon openshift-cert-manager-operator \
  -o jsonpath='state={.status.state}  installedCSV={.status.installedCSV}{"\n"}'

# Verify CSV has reached Succeeded phase
oc get csv -n open-cluster-management-agent-addon \
  -l operators.coreos.com/openshift-cert-manager-operator.open-cluster-management-agent-addon \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.phase}{"\n"}{end}'
# Expected: Succeeded

# Verify cert-manager pods are running
oc get pods -n cert-manager

# Switch back to hub
oc config use-context <hub-context-name>
```

**The add-on is fully installed when:**
1. `ManagedClusterAddOn` shows `AVAILABLE: True`
2. Subscription `state` is `AtLatestKnown`
3. CSV `phase` is `Succeeded`
4. cert-manager pods are running on the managed cluster

### 7. Verify ManifestWork Deployment

```bash
# Check ManifestWork resources (one per cluster)
oc get manifestwork -A | grep cert-manager-addon

# Detailed resource status for specific cluster
CLUSTER="cluster1"
oc get manifestwork -n $CLUSTER addon-cert-manager-addon-deploy-0 \
  -o jsonpath='{range .status.resourceStatus.manifests[*]}{.resourceMeta.kind}/{.resourceMeta.name}: {range .conditions[*]}{.type}={.status} {end}{"\n"}{end}'
```

## Cleanup

To remove the addon from all clusters:

```bash
# 1. Delete ClusterManagementAddOn (cascades to ManagedClusterAddOns)
oc delete clustermanagementaddon cert-manager-addon

# 2. Verify ManagedClusterAddOns removed
oc get managedclusteraddon -A | grep cert-manager-addon
# Should return no results after 1-2 minutes

# 3. Remove configuration resources
oc delete placement cert-manager-placement -n open-cluster-management-global-set
oc delete addontemplate cert-manager-addon-template
```

## Support

For issues or questions:
- Check the [README.md](./README.md) for detailed documentation
- Review the [RHACM Add-on Developer Guide](https://github.com/OpenShift-Fleet/agentic-sdlc/blob/main/workflows/rhacm-addon/developer-guide.md) for architecture and gotchas
- Check ACM addon manager logs: `oc logs -n open-cluster-management-hub deployment/cluster-manager-addon-manager-controller`
