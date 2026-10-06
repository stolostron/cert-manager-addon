# Agents

This document describes the AI agents and workflows used to develop and maintain this repository.

## Repository Purpose

This repository contains RHACM add-on manifests for deploying the OpenShift cert-manager operator to managed clusters via the Template-based add-on framework (OLM-only pattern, no agent Deployment).

## Agent Workflows

### Manifest Authoring

- **Tool:** Claude Code
- **Scope:** Generating and updating add-on YAML manifests (`manifests/`), deployment guides, and pre-flight checklists
- **Validation:** Manifests must pass `oc apply --dry-run=server` against a compatible ACM hub before merge

### Manifest Validation

- **Pre-merge:** Dry-run validation against an ACM hub cluster
- **Post-merge:** Deploy to a staging hub and verify `ManagedClusterAddOn` reaches AVAILABLE=True, Subscription state is `AtLatestKnown`, and the operator CSV phase is `Succeeded` on target managed clusters

## CI Workflows

### `.github/workflows/manifest-lint.yml`

Runs on every PR that touches `manifests/` and on push to `main`:

- **yaml-lint:** `yamllint -d relaxed` on all YAML manifests
- **kube-validate:** `kubectl apply --dry-run=client` for core K8s resources (Namespace, Placement); YAML syntax validation for CRD-dependent resources (AddOnTemplate, ClusterManagementAddOn)

## Contributing

When modifying manifests:
1. Follow the numbered file ordering convention (`01-`, `03-`, `04-`, `05-`, `06-`)
2. Ensure resource dependencies are documented in file-level comments
3. Test the full deployment flow on a compatible ACM hub before submitting a PR
4. Verify OLM operator installation end-to-end (Subscription + CSV + pods), not just ManifestWork application
