# Pre-Flight Checklist

Complete these validation steps **before** applying the cert-manager addon to your hub cluster.

## 1. Hub Cluster Access

- [ ] Connected to hub cluster (not managed cluster)
  ```bash
  oc whoami --show-server
  # Should show your hub cluster API endpoint
  ```

- [ ] Cluster-admin permissions on hub cluster
  ```bash
  oc auth can-i '*' '*' --all-namespaces
  # Should return "yes"
  ```

## 2. Managed Clusters Availability

- [ ] Managed clusters are joined and available
  ```bash
  oc get managedcluster
  # All clusters should show AVAILABLE: True
  ```

- [ ] Verify target clusters exist
  ```bash
  # Replace with your actual cluster names
  oc get managedcluster <cluster-name>
  ```

## 3. Cluster Labels Match Placement

- [ ] Target clusters have the `cert-manager=enabled` label
  ```bash
  oc get managedcluster --show-labels | grep cert-manager
  ```

- [ ] Label clusters if needed
  ```bash
  oc label managedcluster <cluster-name> cert-manager=enabled
  ```

## 4. Operator Package Verification (OLM)

- [ ] cert-manager operator package and channel exist in the catalog of every target managed cluster

  The Subscription is installed on managed clusters, so run these checks once per
  target cluster using its actual kubeconfig context. Do not assume the
  ManagedCluster name matches the context name.

  ```bash
  oc --context=<managed-cluster-context> get packagemanifest openshift-cert-manager-operator -n openshift-marketplace
  # Should find the openshift-cert-manager-operator package
  ```

- [ ] Verify channel availability
  ```bash
  oc --context=<managed-cluster-context> get packagemanifest openshift-cert-manager-operator -n openshift-marketplace -o jsonpath='{.status.channels[*].name}'
  # Should include 'stable-v1' (default channel)
  ```

## 5. Namespace Availability

- [ ] Hub namespace exists (auto-created by the global ManagedClusterSet)
  ```bash
  oc get namespace open-cluster-management-global-set
  # Should exist — it is auto-created by the global ManagedClusterSet
  ```

## 6. RHACM Addon Framework

- [ ] Addon framework is installed (part of ACM/MCE)
  ```bash
  oc get crd clustermanagementaddons.addon.open-cluster-management.io
  oc get crd managedclusteraddons.addon.open-cluster-management.io
  oc get crd addontemplates.addon.open-cluster-management.io
  # All should exist
  ```

- [ ] Addon manager controller is running
  ```bash
  oc get deployment -n open-cluster-management-hub cluster-manager-addon-manager-controller
  # Should show READY (e.g., 3/3)
  ```

## 7. Resource Conflicts

- [ ] No existing addon with same name
  ```bash
  oc get clustermanagementaddon cert-manager-addon
  # Should return "not found"
  ```

- [ ] No conflicting placements
  ```bash
  oc get placement cert-manager-placement -n open-cluster-management-global-set
  # Should return "not found"
  ```

## 8. No Existing cert-manager Installation

- [ ] cert-manager is NOT already installed via policy or manual deployment on target clusters
  ```bash
  # Check on managed clusters (if you have direct access):
  oc get subscription -A --context=<cluster> | grep cert-manager
  oc get csv -A --context=<cluster> | grep cert-manager
  # Both should return empty — if cert-manager is already installed, remove it first
  ```

## 9. Network Connectivity (for managed clusters)

- [ ] Managed clusters can pull from Red Hat operator registry
  ```bash
  # On managed cluster (or verify pull secrets exist):
  oc get secret -n openshift-marketplace | grep redhat
  ```

## 10. YAML Files Ready

- [ ] All 5 YAML files are present in current directory:
  ```bash
  ls -1 *.yaml
  # Should list:
  # - 01-namespace.yaml
  # - 03-placement.yaml
  # - 04-addontemplate.yaml
  # - 05-clustermanagementaddon.yaml
  # - 06-managedclusteraddon.yaml (reference only, not applied)
  ```

- [ ] YAML files have been reviewed and customized:
  - [ ] Placement selectors match your cluster labels
  - [ ] Operator package/channel are correct
  - [ ] Namespace names are appropriate
  - [ ] Resource limits are suitable

## Ready to Deploy?

If all checkboxes above are checked, proceed to deployment:

```bash
# Follow the ordered deployment in DEPLOY.md
cat DEPLOY.md
```
