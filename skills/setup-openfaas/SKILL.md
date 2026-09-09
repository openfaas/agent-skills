---
name: setup-openfaas
description: "Installs, configures, verifies, and upgrades OpenFaaS on Kubernetes with the official Helm chart, or performs a basic OpenFaaS Edge installation on a dedicated Linux host. Supports Community Edition, Standard, For Enterprises, Edge, the Pro dashboard, IAM/SSO, the faas-cli Pro plugin, and Function Builder. Use when asked to set up or operate OpenFaaS or OpenFaaS Edge."
---

# Setup OpenFaaS

Install OpenFaaS into an existing Kubernetes cluster with the official Helm chart, or install OpenFaaS Edge onto a dedicated Linux host. Keep Kubernetes and Edge workflows separate: Edge is a single-host systemd/containerd appliance and does not run on Kubernetes.

## Select the workflow

Determine the edition before changing the target cluster or host. Do not silently default to Community Edition:

- **Community Edition (CE)** is intended for personal exploration; commercial evaluation is time-limited. Read [references/community.md](references/community.md).
- **OpenFaaS Standard** is the production, single-team/single-tenant Pro distribution. Read [references/standard-enterprise.md](references/standard-enterprise.md).
- **OpenFaaS for Enterprises** adds multi-tenancy and optional IAM/SSO. Read [references/standard-enterprise.md](references/standard-enterprise.md), and read [references/iam-sso.md](references/iam-sso.md) when IAM, OIDC, SSO, multiple teams, or multiple function namespaces are requested.
- **OpenFaaS Edge (faasd-pro)** is the commercial, single-host distribution for VMs, bare metal, on-premises appliances, and redistribution to customer sites. It does not use Kubernetes or Helm. Read [references/edge.md](references/edge.md).

For Standard or For Enterprises on Kubernetes, also read [references/pro-cli.md](references/pro-cli.md) to select Basic Auth or IAM authentication and install the Pro plugin only when a plugin feature is required.

For a Kubernetes installation, when the Function Builder is explicitly requested, read both [references/function-builder.md](references/function-builder.md) and [references/local-k3s-registry.md](references/local-k3s-registry.md). The bundled local-registry workflow supports only an unauthenticated, single-node K3s evaluation or development cluster. Verify current Builder license entitlement before deployment; do not infer it from `openfaasPro: true` alone. Edge Function Builder is a different workflow and is outside the basic Edge installation; follow the current official Edge Builder documentation only when explicitly requested.

Read [references/operations.md](references/operations.md) when verifying, upgrading, troubleshooting, using GitOps, or preparing a production Kubernetes installation. Use the verification and troubleshooting workflow in [references/edge.md](references/edge.md) for Edge.

If the user has not identified the edition and it cannot be inferred from an existing installation or license, ask whether they need CE, Standard, For Enterprises, or Edge before installing. Do not select Edge merely because the host is described as an edge device; select it only when a single-host, non-Kubernetes installation is intended.

## Collect inputs

Resolve these from the request and environment; ask only for required choices that remain unknown:

- edition: CE, Standard, For Enterprises, or Edge
- deployment intent: local evaluation/staging, production, or redistribution to customer sites
- for Kubernetes: kubeconfig/context, target cluster, directory for the minimal values file, dashboard choice, and any ingress, TLS, DNS, GitOps, air-gap, or external NATS requirements
- for Standard or For Enterprises: cluster license path, normally `~/.openfaas/LICENSE`
- for Edge: SSH target or local host, Linux distribution and architecture, dedicated-host confirmation, sizing, installation license path, outbound registry access, and whether the gateway must be exposed beyond localhost
- for Kubernetes IAM: gateway and dashboard URLs, OIDC authority, client ID, optional client secret, scopes, and intended users/teams
- for Kubernetes Function Builder: explicit Builder intent, single-node K3s confirmation, K3s server-node shell access, restart impact, Builder values-file path, and restrictive payload-secret client path

Treat the Kubernetes cluster license (`LICENSE`), Edge installation license, and Pro CLI license (`LICENSE_CLI`) as separate credentials unless OpenFaaS explicitly supplied one credential for multiple purposes. Never print licenses, passwords, client secrets, private keys, or tokens.

For Kubernetes, choose one canonical values-file path per Helm release, create parent directories with restrictive permissions, and keep each file mode `0600`. The core `openfaas` and optional `pro-builder` releases use separate values files. Do not leave alternate or intermediate values files elsewhere on the host or cluster node.

## Kubernetes Helm workflow

Use this section only for CE, Standard, or For Enterprises on Kubernetes. For Edge, use [references/edge.md](references/edge.md) and do not create Kubernetes namespaces, add the Helm repository, or install the OpenFaaS chart.

1. Confirm tools and cluster identity before mutation:

   ```bash
   command -v helm kubectl
   kubectl config current-context
   kubectl cluster-info
   kubectl get nodes
   ```

   Standard and Enterprise license inspection also requires `jq` and `base64`. If no cluster exists, stop and use an appropriate cluster-provisioning workflow such as k3sup rather than improvising a different Kubernetes distribution.

2. Inspect an existing installation before deciding whether this is a new install or upgrade:

   ```bash
   helm status openfaas -n openfaas
   helm get values openfaas -n openfaas
   ```

   A `release: not found` result is expected for a new installation. Do not overwrite or discard an existing values file or secret.

3. Create the namespaces and update the official chart repository:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
   helm repo add openfaas https://openfaas.github.io/faas-netes/ --force-update
   helm repo update openfaas
   ```

4. Follow the selected edition reference to create a minimal values file and any required secrets. Before deployment, render and inspect the release:

   ```bash
   OPENFAAS_RENDERED=$(mktemp)
   chmod 600 "$OPENFAAS_RENDERED"
   helm template openfaas openfaas/openfaas \
     --namespace openfaas \
     -f <values-file> > "$OPENFAAS_RENDERED"
   # Inspect only targeted fields, then remove the temporary file.
   rm -f "$OPENFAAS_RENDERED"
   unset OPENFAAS_RENDERED
   ```

   Omit `-f` for CE when there are no overrides. Confirm the expected edition-specific Deployments and CRDs are present, referenced Secret names match the pre-created Secrets, and no credential value is embedded. Inspect targeted kinds, names, images, and `secretName` fields instead of reading thousands of rendered lines. Never put secret values into the values file, and do not retain or share rendered output that contains a Secret.

5. Deploy with Helm:

   ```bash
   helm upgrade --install openfaas openfaas/openfaas \
     --namespace openfaas \
     -f <values-file>
   ```

6. Follow [references/operations.md](references/operations.md) for bounded, action-based readiness and workload, API, CRD, dashboard, and configuration checks. Follow [references/pro-cli.md](references/pro-cli.md) for authentication. IAM-enabled installations must use `faas-cli pro auth`, not the Basic Auth login path.

7. After the core installation is healthy, follow the Function Builder references when it was explicitly requested. Complete the local registry and K3s containerd configuration before installing the `pro-builder` release.

8. Report the edition, Kubernetes context, chart/app versions, values-file paths, applied overrides, gateway/dashboard URLs, authentication method, verification results, and exact Helm commands. Include Builder and registry results when selected. Do not report secret values.

## Safety and scope

- Ask before replacing or rotating an existing license, signing key, AES key, Basic Auth secret, or OAuth client secret. Key rotation can invalidate sessions or interrupt access.
- Generate private material in a restrictive temporary directory and remove it after the Kubernetes Secret is successfully created. Do not use predictable filenames in the working tree.
- Prefer idempotent `kubectl create ... --dry-run=client -o yaml | kubectl apply -f -` only when updating that secret is intentional. Otherwise detect the existing secret and preserve it.
- Do not uninstall OpenFaaS as part of an upgrade.
- Never install Edge alongside Docker or Kubernetes on the same host. Do not replace an existing Docker/containerd/CNI installation, overwrite `/var/lib/faasd`, or reinstall an existing Edge host as though it were new.
- Treat the local registry profile as evaluation/development only. Keep it ClusterIP-only, do not add authentication or expose it, preserve existing K3s registry entries, and account for the single-node restart.
- Ingress, TLS/DNS, external NATS, event connectors, dedicated queue-workers, air-gap mirroring, IAM Policies/Roles, and CI identity federation are adjacent workflows. Configure them only when requested; use the official pages in the relevant reference.

## Authoritative sources

Consult current official documentation before using chart values or commands that may have changed:

- [Deployment overview](https://docs.openfaas.com/deployment/)
- [OpenFaaS Edge deployment](https://docs.openfaas.com/deployment/edge/)
- [OpenFaaS Edge overview](https://docs.openfaas.com/edge/overview/)
- [faasd repository and Edge installer](https://github.com/openfaas/faasd)
- [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas)
- [Standard and For Enterprises installation](https://docs.openfaas.com/deployment/pro/)
- [faas-cli installation and Pro plugin](https://docs.openfaas.com/cli/install/)
- [Function Builder API](https://docs.openfaas.com/openfaas-pro/builder/)
- [Function Builder Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder)

Prefer OpenFaaS documentation and official OpenFaaS GitHub repositories over third-party examples.
