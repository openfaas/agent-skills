---
name: setup-openfaas
description: "Installs, configures, verifies, and upgrades OpenFaaS Community Edition, Standard, or For Enterprises on Kubernetes with Helm. Use for Kubernetes deployment, IAM/SSO, license management, Function Builder, optional network isolation, production readiness, and troubleshooting; use the dedicated Edge skill for single-host deployments."
---

# Setup OpenFaaS on Kubernetes

Install and operate OpenFaaS CE, Standard, or For Enterprises with the official Helm chart.

## Boundaries

Before changing anything, confirm the Kubernetes context, target cluster, edition, and whether the request is an install, upgrade, license replacement, or troubleshooting task. Infer these from an existing release or license when safe; otherwise ask. Preserve existing values, workloads, and Secrets unless replacement is explicit. Never expose credentials or silently add adjacent infrastructure.

Select the edition from the intended use rather than defaulting silently:

- CE is for personal exploration; commercial evaluation is time-limited.
- Standard is the production single-team/single-tenant Pro distribution.
- For Enterprises adds multi-tenancy and optional IAM/SSO.

Collect only inputs needed for the selected path: kubeconfig/context, release values location, license path for Pro editions, authentication mode, and requested ingress, TLS, DNS, GitOps, air-gap, or external NATS choices. Treat those last items as separate scope. Choose one canonical values file per release, create its parent directory restrictively, and keep the file mode `0600`; do not leave intermediate copies.

- For CE, read [references/community.md](references/community.md).
- For Standard or For Enterprises, read [references/standard-enterprise.md](references/standard-enterprise.md).
- For IAM, OIDC, SSO, or multiple teams/namespaces, also read [references/iam-sso.md](references/iam-sso.md).
- For CLI installation and authentication, read [references/pro-cli.md](references/pro-cli.md).
- For upgrades, verification, production review, or troubleshooting, read [references/operations.md](references/operations.md).
- For a single-host non-Kubernetes installation, use `setup-openfaas-edge`.
- When Function Builder is requested, finish and verify the core installation, then read [references/function-builder.md](references/function-builder.md). It contains the single-node K3s local-registry evaluation workflow; do not load it for ordinary setup.
- Network isolation is a strictly optional add-on: apply it only when explicitly requested, after the core installation is healthy. Read [references/network-isolation.md](references/network-isolation.md) for CNI/enforcement checks, the policy, compatibility limits, verification, and rollback. When provisioning a new cluster with isolation requested, check enforcement prerequisites before installing OpenFaaS to avoid a later CNI migration. Do not add isolation during ordinary installation, upgrades, IAM setup, or production review. Preserve existing policies on upgrades.
- If requested isolation lacks a working NetworkPolicy enforcer on K3s, warn that applying a manifest alone provides no protection. Read [references/k3s-cni-migration.md](references/k3s-cni-migration.md) for separate CNI replacement notes. Isolation authorization does not by itself authorize a disruptive CNI migration; stock K3s with a working embedded policy controller needs no CNI swap.
- After the platform is healthy, use `openfaas-function-dev` for building, configuring, deploying, invoking, or troubleshooting functions.

Cluster `LICENSE` and CLI `LICENSE_CLI` are distinct credentials unless OpenFaaS explicitly supplied one for both purposes. License replacement is a separate authorized operation, not part of an ordinary install or upgrade.

For Standard and Enterprise, never create a Secret directly from a possibly annotated multi-line license: normalize one JWT line first and validate the selected product and status without exposing identity claims. Keep operator leader election enabled with multiple gateway replicas. Keep `clusterRole: true` when node metrics, CPU autoscaling, or multiple function namespaces require cluster-wide access.

## Helm workflow

1. Confirm tools and cluster identity:

   ```bash
   command -v helm kubectl
   kubectl config current-context
   kubectl cluster-info
   kubectl get nodes
   ```

   If no cluster exists, stop and use an appropriate cluster-provisioning workflow; do not improvise a Kubernetes distribution.

2. Inspect current state:

   ```bash
   helm status openfaas -n openfaas
   helm get values openfaas -n openfaas
   kubectl get secrets -n openfaas
   ```

   A missing release is normal for a new install. For an existing release, retain its canonical values file and inspect changes before upgrading. For Standard or Enterprise, [check the current static cluster license](references/standard-enterprise.md#check-current-license-status-for-an-installation).

3. Create namespaces and update the official repository:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
   helm repo add openfaas https://openfaas.github.io/faas-netes/ --force-update
   helm repo update openfaas
   ```

4. Follow the edition reference to prepare only the required Secrets and a minimal values file. Keep values files mode `0600`; never embed secret values. Preserve referenced Secrets unless rotation or replacement was requested.

5. Render before deployment:

   ```bash
   OPENFAAS_RENDERED=$(mktemp)
   chmod 600 "$OPENFAAS_RENDERED"
   helm template openfaas openfaas/openfaas \
     --namespace openfaas -f <values-file> > "$OPENFAAS_RENDERED"
   ```

   Omit `-f` for CE without overrides. Inspect only relevant kinds, images, RBAC, security contexts, and Secret references. Confirm that Standard/Enterprise settings include required operator, leader-election, and cluster-wide RBAC choices. Remove the rendered file immediately; do not retain output containing Secret objects.

6. Deploy without uninstalling:

   ```bash
   rm -f "$OPENFAAS_RENDERED"
   unset OPENFAAS_RENDERED
   helm upgrade --install openfaas openfaas/openfaas \
     --namespace openfaas -f <values-file>
   ```

   Omit `-f <values-file>` for CE when there are no overrides. An upgrade must retain deliberate values and use `helm upgrade --install`; never uninstall first. If Helm reports an RBAC ownership conflict while enabling `clusterRole`, remove only the exact object proven to belong to this release and only when required for the upgrade.

7. Use the bounded authenticated checks in [references/operations.md](references/operations.md). Success requires healthy workloads and a successful authenticated `faas-cli list`, not an unauthenticated HTTP response. IAM installations authenticate with `faas-cli pro auth`; the others use Basic Auth.

Report the edition, context, versions, retained values path, intentional overrides, non-secret endpoints, authentication method, verification results, and unresolved in-scope issues.

## Sources

Use current [deployment documentation](https://docs.openfaas.com/deployment/), [Pro deployment guidance](https://docs.openfaas.com/deployment/pro/), and the [official Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas). Prefer official OpenFaaS sources over third-party examples.
