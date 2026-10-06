# cert-manager-addon

RHACM add-on that deploys the **OpenShift cert-manager operator** to managed clusters via the Template-based add-on framework, enabling fleet-wide certificate lifecycle management.

## Overview

This add-on uses the **OLM-only** pattern (no agent Deployment). It installs `openshift-cert-manager-operator` on managed clusters via an OLM Subscription, with placement-based cluster selection and progressive rollout.

### What Gets Deployed to Managed Clusters

- Aggregating ClusterRole (grants work-agent OperatorGroup RBAC)
- Namespace for OLM resources
- OperatorGroup
- OLM Subscription for `openshift-cert-manager-operator` (channel `stable-v1`)

## Quick Start

```bash
# Label target clusters
oc label managedcluster <cluster-name> cert-manager=enabled

# Apply manifests in order
oc apply -f manifests/01-namespace.yaml
oc apply -f manifests/03-placement.yaml
oc apply -f manifests/04-addontemplate.yaml
oc apply -f manifests/05-clustermanagementaddon.yaml

# Monitor rollout (AVAILABLE=True means ManifestWork applied, not OLM install complete)
watch -n 5 'oc get managedclusteraddon -A | grep cert-manager-addon'

# Verify OLM operator is fully installed (check per-cluster Subscription/CSV status)
# See manifests/DEPLOY.md step 6 for details
```

## Documentation

| Document | Description |
|----------|-------------|
| [manifests/README.md](manifests/README.md) | Architecture, configuration, and troubleshooting |
| [manifests/DEPLOY.md](manifests/DEPLOY.md) | Step-by-step deployment guide |
| [manifests/PRE_FLIGHT_CHECKLIST.md](manifests/PRE_FLIGHT_CHECKLIST.md) | Pre-deployment validation checklist |
| [docs/migration-guide.md](docs/migration-guide.md) | Migrating existing cert-manager installs to the add-on |
| [docs/issuer-guides.md](docs/issuer-guides.md) | Issuer and ClusterIssuer configuration examples |

## Related

- **Jira:** [ACM-43686](https://redhat.atlassian.net/browse/ACM-43686) — Implement cert-manager add-on in dedicated stolostron repository
- **Parent Epic:** [ACM-42688](https://redhat.atlassian.net/browse/ACM-42688) — cert-manager ACM add-on for fleet-wide certificate management
- **RFE:** [RFE-9771](https://redhat.atlassian.net/browse/RFE-9771) — Provide cert-manager as a native ACM add-on
- **Developer Guide:** [RHACM Add-on Developer Guide](https://github.com/OpenShift-Fleet/agentic-sdlc/blob/main/workflows/rhacm-addon/developer-guide.md)
