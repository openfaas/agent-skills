---
name: setup-openfaas
description: "Installs, configures, verifies, and upgrades OpenFaaS on an existing Kubernetes cluster with the official Helm chart. Supports Community Edition, OpenFaaS Standard, OpenFaaS for Enterprises, the Pro dashboard, IAM/SSO, and the faas-cli Pro plugin. Use when asked to set up or operate an OpenFaaS cluster."
---

# Setup OpenFaaS

Install OpenFaaS into an existing Kubernetes cluster with the official Helm chart. Keep generated Helm overrides minimal and retain them for upgrades and support.

## Select the workflow

Determine the edition before changing the cluster. Do not silently default to Community Edition:

- **Community Edition (CE)** is intended for personal exploration; commercial evaluation is time-limited. Read [references/community.md](references/community.md).
- **OpenFaaS Standard** is the production, single-team/single-tenant Pro distribution. Read [references/standard-enterprise.md](references/standard-enterprise.md).
- **OpenFaaS for Enterprises** adds multi-tenancy and optional IAM/SSO. Read [references/standard-enterprise.md](references/standard-enterprise.md), and read [references/iam-sso.md](references/iam-sso.md) when IAM, OIDC, SSO, multiple teams, or multiple function namespaces are requested.

For Standard or For Enterprises, also read [references/pro-cli.md](references/pro-cli.md) to install and activate the Pro plugin and select Basic Auth or IAM authentication correctly.

Read [references/operations.md](references/operations.md) when verifying, upgrading, troubleshooting, using GitOps, or preparing a production installation.

If the user has not identified the edition and it cannot be inferred from an existing release or license, ask whether they need CE, Standard, or For Enterprises before installing.

## Collect inputs

Resolve these from the request and environment; ask only for required choices that remain unknown:

- edition: CE, Standard, or For Enterprises
- kubeconfig/context and target cluster
- directory for the minimal values file
- Standard/Enterprise cluster license path, normally `~/.openfaas/LICENSE`
- dashboard enabled or disabled
- any ingress, TLS, DNS, GitOps, air-gap, or external NATS requirements
- for IAM: gateway and dashboard URLs, OIDC authority, client ID, optional client secret, scopes, and intended users/teams

Treat the cluster license (`LICENSE`) and Pro CLI license (`LICENSE_CLI`) as separate credentials. Never print licenses, passwords, client secrets, private keys, or tokens.

Choose one canonical values-file path, create parent directories with restrictive permissions, and keep that file mode `0600`. Do not leave alternate or intermediate values files elsewhere on the host or cluster node.

## Common Helm workflow

1. Confirm tools and cluster identity before mutation:

   ```bash
   command -v helm kubectl
   kubectl config current-context
   kubectl cluster-info
   kubectl get nodes
   ```

2. Inspect an existing installation before deciding whether this is a new install or upgrade:

   ```bash
   helm status openfaas -n openfaas
   helm get values openfaas -n openfaas
   ```

   A `release: not found` result is expected for a new installation. Do not overwrite or discard an existing values file or secret.

3. Create the namespaces and update the official chart repository:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
   helm repo add openfaas https://openfaas.github.io/faas-netes/
   helm repo update
   ```

4. Follow the selected edition reference to create a minimal values file and any required secrets. Before deployment, render and inspect the release:

   ```bash
   OPENFAAS_RENDERED=$(mktemp)
   chmod 600 "$OPENFAAS_RENDERED"
   helm template openfaas openfaas/openfaas \
     --namespace openfaas \
     -f <values-file> > "$OPENFAAS_RENDERED"
   # Inspect the rendered resources, then remove the temporary file.
   rm -f "$OPENFAAS_RENDERED"
   unset OPENFAAS_RENDERED
   ```

   Omit `-f` for CE when there are no overrides. Never put secret values into the values file, and do not retain or share rendered output that contains a Secret.

5. Deploy with Helm:

   ```bash
   helm upgrade --install openfaas openfaas/openfaas \
     --namespace openfaas \
     -f <values-file> \
     --wait \
     --timeout 10m
   ```

6. Follow [references/operations.md](references/operations.md) for workload, API, CRD, dashboard, and configuration checks. Follow [references/pro-cli.md](references/pro-cli.md) for authentication. IAM-enabled installations must use `faas-cli pro auth`, not the Basic Auth login path.

7. Report the edition, Kubernetes context, chart/app versions, values-file path, applied overrides, gateway/dashboard URLs, authentication method, verification results, and exact Helm command. Do not report secret values.

## Safety and scope

- Ask before replacing or rotating an existing license, signing key, AES key, Basic Auth secret, or OAuth client secret. Key rotation can invalidate sessions or interrupt access.
- Generate private material in a restrictive temporary directory and remove it after the Kubernetes Secret is successfully created. Do not use predictable filenames in the working tree.
- Prefer idempotent `kubectl create ... --dry-run=client -o yaml | kubectl apply -f -` only when updating that secret is intentional. Otherwise detect the existing secret and preserve it.
- Do not uninstall OpenFaaS as part of an upgrade.
- Ingress, TLS/DNS, external NATS, event connectors, dedicated queue-workers, Function Builder, air-gap mirroring, IAM Policies/Roles, and CI identity federation are adjacent workflows. Configure them only when requested; use the official pages in the relevant reference.

## Authoritative sources

Consult current official documentation before using chart values or commands that may have changed:

- [Deployment overview](https://docs.openfaas.com/deployment/)
- [OpenFaaS Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas)
- [Standard and For Enterprises installation](https://docs.openfaas.com/deployment/pro/)
- [faas-cli installation and Pro plugin](https://docs.openfaas.com/cli/install/)

Prefer OpenFaaS documentation and official OpenFaaS GitHub repositories over third-party examples.
