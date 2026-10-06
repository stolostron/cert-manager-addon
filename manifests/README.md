# cert-manager Add-on for RHACM

Deploys the **OpenShift cert-manager operator** to managed clusters via RHACM's Template-based add-on framework, enabling fleet-wide certificate lifecycle management.

## Architecture

```
Hub Cluster:
├── ClusterManagementAddOn (cert-manager-addon)
│   ├── installStrategy: Placements → cert-manager-placement
│   └── supportedConfigs → cert-manager-addon-template
├── AddOnTemplate (cert-manager-addon-template)
│   ├── Manifests: Aggregating ClusterRole, Namespace, OperatorGroup, OLM Subscription
│   └── Feedback: Subscription state + installedCSV (surfaced via ManifestWork)
└── Placement (cert-manager-placement)
    └── Selector: cert-manager=enabled

Managed Clusters (auto-created by addon-manager):
└── ManagedClusterAddOn (cert-manager-addon)
    └── ManifestWork → deploys cert-manager operator via OLM
```

## Quick Start

```bash
# 1. Label target clusters
oc label managedcluster <cluster-name> cert-manager=enabled

# 2. Deploy (follow order!)
oc apply -f 01-namespace.yaml
oc apply -f 03-placement.yaml
oc apply -f 04-addontemplate.yaml
oc apply -f 05-clustermanagementaddon.yaml

# 3. Monitor (AVAILABLE=True = ManifestWork applied; verify OLM separately)
watch -n 5 'oc get managedclusteraddon -A | grep cert-manager-addon'
```

See **DEPLOY.md** for complete deployment guide with troubleshooting.

## Files

| File | Purpose |
|------|---------|
| `01-namespace.yaml` | Hub namespace (apply first) |
| `03-placement.yaml` | Cluster selection (cert-manager=enabled) |
| `04-addontemplate.yaml` | Workload manifests (OLM Subscription + RBAC) |
| `05-clustermanagementaddon.yaml` | Main add-on definition (triggers rollout) |
| `06-managedclusteraddon.yaml` | Per-cluster instance (REFERENCE ONLY — auto-created) |
| `DEPLOY.md` | Ordered deployment commands |
| `PRE_FLIGHT_CHECKLIST.md` | Validation before apply |

## Configuration

### Cluster Selection

Edit `03-placement.yaml` to change which clusters receive cert-manager:

```yaml
predicates:
  - requiredClusterSelector:
      labelSelector:
        matchLabels:
          cert-manager: enabled    # ← Change to your selector
```

To opt in a cluster: `oc label managedcluster <name> cert-manager=enabled`

### Rollout Strategy

Progressive rollout: 2 clusters at a time, 5 min wait between batches.

To change: Edit `05-clustermanagementaddon.yaml` rolloutStrategy section.

## Troubleshooting

### Addon shows DEGRADED

```bash
CLUSTER="cluster1"
oc get managedclusteraddon -n $CLUSTER cert-manager-addon -o jsonpath='{.status.conditions}' | jq .
```

**Common causes:**
- AddOnTemplate not referenced in supportedConfigs
- Health check resource doesn't exist on managed cluster
- cert-manager operator package not available in catalog

### Placement selects zero clusters

```bash
oc get placementdecision -n open-cluster-management-global-set \
  -l cluster.open-cluster-management.io/placement=cert-manager-placement
```

**Fix:** Ensure clusters are labeled: `oc label managedcluster <name> cert-manager=enabled`

### cert-manager operator not installing on managed cluster

```bash
# Check Subscription status (on managed cluster — namespace overridden by addon framework)
oc get subscription -n open-cluster-management-agent-addon openshift-cert-manager-operator -o yaml --context=<cluster>

# Check if package exists in catalog
oc get packagemanifest openshift-cert-manager-operator -n openshift-marketplace --context=<cluster>
```

### View addon manager logs (hub)

```bash
oc logs -n open-cluster-management-hub deployment/cluster-manager-addon-manager-controller
```

## Cleanup

```bash
# Remove addon (cascades to all clusters)
oc delete clustermanagementaddon cert-manager-addon

# Wait for ManagedClusterAddOns to be removed
oc get managedclusteraddon -A | grep cert-manager-addon

# Remove config resources
oc delete placement cert-manager-placement -n open-cluster-management-global-set
oc delete addontemplate cert-manager-addon-template
```

**Note:** This does NOT uninstall cert-manager from managed clusters. To fully clean up:

```bash
# On each managed cluster (namespace overridden by addon framework):
oc delete subscription openshift-cert-manager-operator -n open-cluster-management-agent-addon --context=<cluster>
oc delete csv -n open-cluster-management-agent-addon -l operators.coreos.com/openshift-cert-manager-operator.open-cluster-management-agent-addon --context=<cluster>
```

## Related

- **RFE:** [RFE-9771](https://redhat.atlassian.net/browse/RFE-9771) — Provide cert-manager as a native ACM add-on (Approved)
- **Feature:** [ACM-42773](https://redhat.atlassian.net/browse/ACM-42773) — cert-manager fleet deployment as a native ACM managed cluster add-on
- **Epic:** [ACM-42688](https://redhat.atlassian.net/browse/ACM-42688) — cert-manager ACM add-on for fleet-wide certificate management
- **Developer Guide:** [RHACM Add-on Developer Guide](https://github.com/OpenShift-Fleet/agentic-sdlc/blob/main/workflows/rhacm-addon/developer-guide.md)
