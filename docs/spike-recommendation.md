# ACM-42782: Spike Recommendation — cert-manager Fleet Deployment via OCM AddOnTemplate

**Spike:** [ACM-42782](https://redhat.atlassian.net/browse/ACM-42782)
**Feature:** [ACM-42773](https://redhat.atlassian.net/browse/ACM-42773)
**Epic:** [ACM-42688](https://redhat.atlassian.net/browse/ACM-42688)
**RFE:** [RFE-9771](https://redhat.atlassian.net/browse/RFE-9771)
**Date:** 2026-09-11

---

## 1. Executive Summary

The OCM AddOnTemplate framework is a viable and recommended approach for delivering cert-manager as a native ACM managed-cluster add-on. The proof-of-concept implementation in `stolostron/cert-manager-addon` ([PR #1](https://github.com/stolostron/cert-manager-addon/pull/1)) defines deployment of `openshift-cert-manager-operator` via OLM Subscription and configures Subscription status as ManifestWork feedback for post-apply verification. The add-on availability condition reflects ManifestWork application; it is not gated on OLM resolution. The OLM-only pattern (no agent Deployment, no custom Go code) minimizes maintenance burden and aligns with existing ACM add-on precedent (e.g., pipelines-operator). We recommend proceeding with implementation, scoped to four stories detailed in Section 10.

---

## 2. Investigation Area 1: AddOnTemplate Feasibility

### What the PoC Proved

The proof-of-concept validates the following capabilities:

| Capability | Status | Notes |
|---|---|---|
| OLM operator deployment via AddOnTemplate | Validated | Subscription + OperatorGroup + Namespace via ManifestWork |
| No agent Deployment required | Validated | Work-agent applies manifests; OLM handles operator lifecycle |
| Placement-based cluster selection | Validated | Label selector (`cert-manager=enabled`) |
| Progressive rollout | Validated | `installStrategy.type: Placements` with configurable concurrency |
| Subscription status feedback surfaced to hub | Configured | `feedbackRules` expose `installedCSV` and `state` through ManifestWork status; verify end-to-end on a staging hub |
| CI validation | Validated | GitHub Actions workflow with yamllint + kubectl dry-run |

### Closest Reference Add-on

The `pipelines-operator` add-on is the closest analog in the ACM ecosystem. Both deploy an OLM operator to managed clusters via AddOnTemplate with no custom controller. The cert-manager add-on follows the same pattern with cert-manager-specific operator package, channel, and RBAC.

### ClusterSet-Scoped Deployment

One-step deployment to a ClusterSet is supported natively via the Placement API. No net-new plumbing is required. The Placement resource accepts a `clusterSets` field:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: cert-manager-placement
  namespace: open-cluster-management-global-set
spec:
  clusterSets:
    - production           # deploys to all clusters in the "production" ClusterSet
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            vendor: OpenShift
```

This satisfies ACM-42773's acceptance criterion: "cert-manager deployable to all managed clusters or a ClusterSet in one step."

### Framework Gaps Identified

| Gap | Impact | Mitigation |
|---|---|---|
| OLM status does not gate add-on availability | `ManagedClusterAddOn` availability reflects ManifestWork application, not Subscription or CSV status | Verify Subscription state and CSV phase separately on each managed cluster; use ManifestWork feedback for diagnosis |
| No cert-level visibility | ManifestWork and OLM status feedback do not report individual certificate status | Addressed by Tier 1 (RFE-9770); not a blocker for Tier 2 |

---

## 3. Investigation Area 2: Deployment Model

### Minimum Install Footprint

The add-on deploys four resources to each managed cluster via ManifestWork:

| Resource | Purpose |
|---|---|
| ClusterRole (aggregating) | Grants work-agent permission to create OperatorGroups (`aggregate-to-work: "true"`) |
| Namespace | Target namespace for OLM resources (overridden to `installNamespace` by addon framework) |
| OperatorGroup | Required for OLM to process Subscriptions in the install namespace |
| Subscription | Installs `openshift-cert-manager-operator` from `redhat-operators` catalog, `stable-v1` channel |

Once OLM resolves the Subscription, the cert-manager operator installs its own CRDs (`Certificate`, `Issuer`, `ClusterIssuer`, etc.) and controller pods in the `cert-manager` namespace.

### CA Credential Distribution

This is the primary post-install configuration challenge. Three options were evaluated:

| Option | Mechanism | Pros | Cons |
|---|---|---|---|
| Manual post-install | Admin creates Issuer/ClusterIssuer + Secret per cluster | Simple, no framework changes | Does not scale, requires per-cluster access |
| AddOnDeploymentConfig variables | Pass CA bundle via `customizedVariables`, reference in a ConfigMap within AddOnTemplate | Uses existing framework, per-placement override possible | Variables are plaintext (not suitable for private keys); limited to values referenced in template |
| ManifestWork with Secret | Include a Secret resource in the AddOnTemplate manifests | Deploys CA credentials alongside operator | Secret is in plaintext in the ManifestWork on the hub; same CA for all clusters in the placement |

**Recommendation:** For the initial release, document manual post-install Issuer configuration (Option 1) with guided documentation for common patterns. CA credential distribution via AddOnDeploymentConfig or a dedicated Secret delivery mechanism should be evaluated as a follow-on story once customer usage patterns are clearer.

See [docs/issuer-guides.md](./issuer-guides.md) for guided Issuer configuration documentation.

---

## 4. Investigation Area 3: Health and Observability

### Current Status and Feedback Architecture

```
Managed Cluster                          Hub Cluster
+----------------------------+           +----------------------------------+
| Subscription               |           | ManifestWork                     |
|   .status.state            |  feedback |   .status.resourceStatus         |
|   .status.installedCSV     | --------> |     .statusFeedback.values       |
+----------------------------+           +----------------------------------+
                                                      |
                                                      v
                                         +----------------------------------+
                                         | ManagedClusterAddOn              |
                                         |   AVAILABLE = ManifestWork status|
                                         |   conditions[]                   |
                                         +----------------------------------+
                                                      |
                                                      v
                                         +----------------------------------+
                                         | ACM Console                      |
                                         |   Add-on status per cluster      |
                                         |   AVAILABLE / DEGRADED / ...     |
                                         +----------------------------------+
```

### What the ACM Console Shows Today

The ACM console displays `ManagedClusterAddOn` conditions. `AVAILABLE=True` indicates that the ManifestWork was applied; it does not establish that OLM resolved the Subscription or that the operator CSV succeeded.

- **AVAILABLE=True**: ManifestWork applied; verify Subscription and CSV status separately on the managed cluster
- **AVAILABLE=False / DEGRADED**: ManifestWork or add-on deployment problem; inspect ManifestWork status and managed-cluster events
- **Progressing**: Rollout in progress (progressive strategy)

The `state` and `installedCSV` values are surfaced via feedbackRules and visible in ManifestWork status, but are not currently displayed in the console UI as first-class fields.

### Gap: Certificate-Level Visibility

The add-on availability condition reflects ManifestWork application, not whether the cert-manager **operator** is installed and running. Subscription feedback can aid OLM troubleshooting, but it does not surface:

- Individual `Certificate` resource status (ready/expired/failing)
- Issuer health (configured/erroring)
- Certificate expiration warnings

This is the domain of Tier 1 ([RFE-9770](https://redhat.atlassian.net/browse/RFE-9770) — fleet-wide certificate visibility).

### Tier 1 Dependency Assessment

**Tier 2 can proceed independently of Tier 1.** The add-on delivers operator deployment and exposes OLM status feedback for verification; its availability condition does not represent operator health. Certificate-level visibility is an orthogonal concern that layers on top. There is no technical dependency — Tier 2 does not consume or require any API or UI from Tier 1.

---

## 5. Investigation Area 4: Ownership and Ecosystem

### Recommended Ownership Model

| Component | Owner | Rationale |
|---|---|---|
| `openshift-cert-manager-operator` | cert-manager operator team | Upstream operator, OLM packaging, catalog entry |
| `stolostron/cert-manager-addon` | Server Foundation | Add-on wrapper (manifests, Placement, rollout strategy, status feedback) |
| ACM console integration | Console team (if UI changes needed) | Status display, future Tier 3 one-click experience |

Server Foundation owns the add-on packaging and lifecycle, not the operator itself. This is consistent with how other OLM-based add-ons are structured — the add-on team does not fork or modify the upstream operator.

### Repository Location

`stolostron/cert-manager-addon` is the correct home. This follows the established pattern:

- `stolostron/` for ACM-specific component repositories
- Dedicated repository per add-on (not bundled in agentic-sdlc or a mono-repo)
- Standard CI, OWNERS, and release tooling

### Upstream Considerations

No upstream OCM contributions are required for the initial implementation. The AddOnTemplate API is stable and sufficient. If future requirements (e.g., cert-level health aggregation) need framework changes, those would be proposed to `open-cluster-management-io/addon-framework`.

---

## 6. Investigation Area 5: Fleet and Multicluster

### Global Hub / Hub-of-Hubs

The AddOnTemplate + Placement pattern should propagate correctly in Global Hub topologies because:

- `ClusterManagementAddOn` is a hub-scoped resource
- Placement decisions are evaluated per hub
- ManifestWorks are created per managed cluster, per hub

**Caveat:** This has not been validated in a Global Hub environment. Validation should be included in the implementation stories (see Story 3 in Section 10).

### Interaction with Policy-Based Installs

Customers who already deploy cert-manager via ACM Policy (e.g., a ConfigurationPolicy creating a Subscription) may experience conflicts:

| Scenario | Risk | Mitigation |
|---|---|---|
| Policy and add-on both create Subscription | Conflicting ownership, update fights | Document: remove Policy-based Subscription before enabling add-on |
| Policy targets different namespace | Two cert-manager instances | Pre-flight checklist already checks for existing installations |
| Customer has custom Issuer configuration | Add-on does not manage Issuers | No conflict — add-on only installs operator, Issuers are separate |

### Migration Path

For customers transitioning from manual or Policy-based cert-manager deployment to the native add-on:

1. Remove Policy-managed Subscription and OperatorGroup (leave CRDs and Issuers intact)
2. Label target clusters (`cert-manager=enabled`)
3. Apply add-on manifests from hub
4. Verify operator re-installs via add-on (existing Issuers and Certificates are preserved)

See [docs/migration-guide.md](./migration-guide.md) for detailed migration procedures.

---

## 7. Investigation Area 6: Dependencies and Sequencing

### Dependency Assessment

| Dependency | Status | Blocking? |
|---|---|---|
| OCM AddOnTemplate API | Stable in ACM 2.x | No |
| `openshift-cert-manager-operator` in `redhat-operators` catalog | Available, `stable-v1` channel | No |
| RFE-9770 (Tier 1 — certificate visibility) | Not started | **No** — Tier 2 is independent |
| ACM console changes | Not required for Tier 2 | No — existing ManagedClusterAddOn display is sufficient |
| `stolostron/cert-manager-addon` repository | Created, PR under review | No |

### Sequencing Recommendation

```
ACM-43686 (repo setup + manifests)     <- current PR, in review
    |
    v
Story 1: Issuer documentation          <- no code dependencies
Story 2: ClusterSet + rollout testing   <- depends on ACM-43686 merge
Story 3: Global Hub validation          <- depends on Story 2
Story 4: Migration guide + pre-flight   <- can parallel with Story 2
```

---

## 8. Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | OLM Subscription failure on managed clusters (missing catalog, network issues) is not reflected in add-on availability | Medium | High | feedbackRules surface Subscription state and `installedCSV` in ManifestWork status; pre-flight checklist validates catalog availability per target cluster and deployment guide documents direct Subscription/CSV checks |
| 2 | Conflict with existing Policy-based cert-manager deployments | Medium | Medium | Pre-flight checklist checks for existing installations; migration guide documents transition path |
| 3 | Global Hub topology not validated | Low | Medium | Scoped as a dedicated validation story; AddOnTemplate pattern is expected to propagate correctly |
| 4 | CA credential distribution does not scale without framework support | Medium | Medium | Initial release uses manual post-install configuration; framework-assisted CA distribution is a follow-on story based on customer feedback |

---

## 9. Effort Estimate

| Story | Size | Rationale |
|---|---|---|
| ACM-43686: Repository setup + manifests | **S** | Largely complete (PR under review) |
| Story 1: Issuer documentation | **S** | Documentation only, no code |
| Story 2: ClusterSet + rollout validation | **M** | Requires ACM hub environment, end-to-end testing |
| Story 3: Global Hub validation | **M** | Requires Global Hub environment, may surface framework gaps |
| Story 4: Migration guide + pre-flight enhancements | **S** | Documentation + minor checklist updates |

**Total estimated effort:** 1 Small (in progress) + 2 Small + 2 Medium

---

## 10. Proposed Implementation Stories

### Story 1: Issuer Configuration Documentation

**Title:** Document common cert-manager Issuer configurations for ACM add-on users

**Scope:** Create guided documentation for three Issuer patterns: Let's Encrypt (ACME HTTP-01 and DNS-01), internal CA (self-signed root + CA Issuer chain), and HashiCorp Vault. Documentation is post-install guidance, not automation.

**Acceptance Criteria:**
- [ ] Issuer guide covers Let's Encrypt ACME with HTTP-01 solver
- [ ] Issuer guide covers internal CA (self-signed root + CA Issuer)
- [ ] Issuer guide covers Vault PKI backend
- [ ] Each guide includes sample YAML and verification steps
- [ ] Guides are linked from the main README and DEPLOY.md

---

### Story 2: ClusterSet-Scoped Deployment and End-to-End Validation

**Title:** Validate cert-manager add-on deployment to ClusterSet with progressive rollout

**Scope:** Test and document ClusterSet-scoped deployment. Validate progressive rollout across multiple clusters. Confirm add-on availability reflects ManifestWork application and verify Subscription/CSV status separately on each target cluster. Confirm the operator installs and cert-manager pods run on all target clusters.

**Acceptance Criteria:**
- [ ] Placement configured with `clusterSets` field targets a specific ClusterSet
- [ ] Progressive rollout (configurable concurrency) validated across 3+ clusters
- [ ] ManifestWork feedback exposes Subscription `state` and `installedCSV`; separately verify Subscription resolution and CSV `Succeeded` on each target cluster
- [ ] feedbackRules values visible in ManifestWork status
- [ ] End-to-end test: Certificate resource created and issued on managed cluster
- [ ] Results documented with test evidence

---

### Story 3: Global Hub Topology Validation

**Title:** Validate cert-manager add-on in Global Hub / Hub-of-Hubs topology

**Scope:** Deploy the cert-manager add-on in a Global Hub environment and validate that AddOnTemplate, Placement, and ManifestWork propagate correctly across hub tiers. Document any framework gaps.

**Acceptance Criteria:**
- [ ] Add-on deployed from Global Hub propagates to leaf hubs
- [ ] ManagedClusterAddOns created on leaf hub managed clusters
- [ ] Health status propagates back to Global Hub
- [ ] Any framework gaps documented and filed as upstream issues

---

### Story 4: Migration Guide and Pre-Flight Enhancements

**Title:** Document migration from Policy-based cert-manager to native ACM add-on

**Scope:** Create migration documentation for customers transitioning from Policy-based or manual cert-manager deployment. Enhance pre-flight checklist with conflict detection.

**Acceptance Criteria:**
- [ ] Migration guide covers: Policy removal, label application, add-on deployment, verification
- [ ] Pre-flight checklist detects existing cert-manager Subscriptions and Policies
- [ ] Migration preserves existing Issuer and Certificate resources
- [ ] Rollback procedure documented

---

### Story 5 (Follow-on): CA Credential Distribution via Add-on Framework

**Title:** Investigate automated CA credential distribution for cert-manager Issuers

**Scope:** Evaluate framework-assisted approaches for distributing CA certificates and keys to managed clusters as part of the add-on deployment. This is a follow-on story contingent on customer demand.

**Acceptance Criteria:**
- [ ] Evaluated: AddOnDeploymentConfig customizedVariables for CA bundle
- [ ] Evaluated: ManifestWork with Secret resource for private keys
- [ ] Evaluated: External secret management integration (e.g., Vault agent)
- [ ] Recommendation documented with security considerations

---

## 11. Gap Analysis vs ACM-42773 Acceptance Criteria

| ACM-42773 Acceptance Criterion | Status | Evidence / Gap |
|---|---|---|
| cert-manager deployable to all managed clusters or a ClusterSet in one step | **Met** | Placement supports both label-based and ClusterSet-scoped selection. Single `oc apply -k` deploys all manifests. Validated in PoC for label-based; ClusterSet requires end-to-end validation (Story 2). |
| Add-on deployment and installation status surfaced in ACM console | **Partially met** | ACM console displays ManifestWork-based ManagedClusterAddOn conditions; feedbackRules expose Subscription state and `installedCSV` in ManifestWork status, but OLM status does not gate availability and these values are not first-class console fields. Cert-level visibility requires Tier 1. |
| Guided documentation for common Issuer configurations | **Not yet met** | Scoped as Story 1. Three Issuer patterns identified (Let's Encrypt, internal CA, Vault). |
| Policy-based deployment not required | **Met** | Add-on framework is the supported path. No Policy resources required. |

---

## Appendix: Reference Links

- **PoC Implementation:** [stolostron/cert-manager-addon PR #1](https://github.com/stolostron/cert-manager-addon/pull/1)
- **Parent Epic:** [ACM-42688](https://redhat.atlassian.net/browse/ACM-42688)
- **Feature:** [ACM-42773](https://redhat.atlassian.net/browse/ACM-42773)
- **Implementation Story:** [ACM-43686](https://redhat.atlassian.net/browse/ACM-43686)
- **RFE:** [RFE-9771](https://redhat.atlassian.net/browse/RFE-9771)
- **Tier 1 Dependency:** [RFE-9770](https://redhat.atlassian.net/browse/RFE-9770)
- **RHACM Add-on Developer Guide:** [agentic-sdlc/workflows/rhacm-addon/developer-guide.md](https://github.com/OpenShift-Fleet/agentic-sdlc/blob/main/workflows/rhacm-addon/developer-guide.md)
