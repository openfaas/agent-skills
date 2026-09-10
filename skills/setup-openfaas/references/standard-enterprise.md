# Standard and For Enterprises

Both editions use the same chart and `openfaasPro: true`; entitlements come from the cluster license. Standard targets a single team and function namespace. For Enterprises adds features such as multiple function namespaces and IAM/SSO.

## Contents

- [Cluster license](#cluster-license)
- [Inspect a cluster license](#inspect-a-cluster-license)
- [Check current license status for an installation](#check-current-license-status-for-an-installation)
- [Replace an existing license](#replace-an-existing-license)
- [Select a deployment profile](#select-a-deployment-profile)
- [Pre-install Secret checklist](#pre-install-secret-checklist)
- [Dashboard signing key](#dashboard-signing-key)
- [Enterprise additions](#enterprise-additions)

## Cluster license

For an existing installation, [check the installed license](#check-current-license-status-for-an-installation); no source license file is needed. Preserve the existing `openfaas-license` Secret unless the user explicitly requested license replacement.

For a new installation or an explicitly requested replacement, prepare the supplied file below.

### Prepare a supplied license file

Verify that the license file exists without displaying it.

Confirm the normalization tools are installed:

```bash
command -v grep sed awk
```

Issued files may contain a comment header and `---` separator instead of a bare JWT. Normalize one JWT line into a restrictive temporary file; never modify the source license:

```bash
OPENFAAS_LICENSE_SOURCE=<cluster-license-path>
OPENFAAS_LICENSE_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_LICENSE_DIR"
OPENFAAS_LICENSE_JWT="$OPENFAAS_LICENSE_DIR/license"
trap 'rm -f "$OPENFAAS_LICENSE_JWT"; rmdir "$OPENFAAS_LICENSE_DIR" 2>/dev/null || true' EXIT
if grep -qx -- '---' "$OPENFAAS_LICENSE_SOURCE"; then
  sed -n '/^---$/,$p' "$OPENFAAS_LICENSE_SOURCE" | sed '1d;/^[[:space:]]*$/d;q' > "$OPENFAAS_LICENSE_JWT"
else
  sed '/^[[:space:]]*$/d;q' "$OPENFAAS_LICENSE_SOURCE" > "$OPENFAAS_LICENSE_JWT"
fi
chmod 600 "$OPENFAAS_LICENSE_JWT"
awk -F. 'NF == 3 { found=1 } END { exit !found }' "$OPENFAAS_LICENSE_JWT"
```

[Inspect the normalized license](#inspect-a-cluster-license) before creating or replacing the Secret. Continue only when the reported entitlement matches the selected edition and the license is currently valid.

For a new installation:

```bash
kubectl create secret generic openfaas-license \
  -n openfaas \
  --from-file license="$OPENFAAS_LICENSE_JWT"
rm -f "$OPENFAAS_LICENSE_JWT"
rmdir "$OPENFAAS_LICENSE_DIR"
trap - EXIT
unset OPENFAAS_LICENSE_SOURCE OPENFAAS_LICENSE_JWT OPENFAAS_LICENSE_DIR
```

Clean the temporary directory on both success and failure. Never create the Secret directly from an unnormalized, multi-line license file.

### Inspect a cluster license

Use this procedure for both a prepared license file and the license retrieved from an existing installation. In an offline or air-gapped environment, use an already installed plugin; if unavailable, use the limited fallback below. Otherwise, ensure the Pro plugin's license command is available:

```bash
if ! faas-cli pro license print --help >/dev/null 2>&1; then
  faas-cli plugin get pro
fi
```

For a file prepared by [the normalization procedure](#prepare-a-supplied-license-file):

```bash
faas-cli pro license print "$OPENFAAS_LICENSE_JWT" |
  awk '/^(Expires in|Products|Status):/'
```

For an installed license, use [the Secret retrieval command](#check-current-license-status-for-an-installation) instead.

For either input, confirm that `Products` includes the expected OpenFaaS entitlement and inspect `Status` rather than relying only on the command's exit code. `Expires in` reports the remaining lifetime and expiration date. Report only entitlement and expiration/status information; unfiltered output may contain identity metadata.

`license print` checks format, supported product, and temporal claim status, but does not verify the signature. Do not use `license validate` for a cluster license because it requires the separate `openfaas-cli` entitlement.

When the plugin is unavailable and cannot be downloaded, decode only the entitlement and expiry claims. For a prepared license file:

```bash
command -v base64 jq
cut -d. -f2 "$OPENFAAS_LICENSE_JWT" | tr '_-' '/+' \
  | awk '{m=length($0)%4; if(m==2)$0=$0"=="; else if(m==3)$0=$0"="; print}' \
  | base64 -d | jq '{products, exp}'
```

For an installed license, retrieve and decode the Secret directly without the Pro plugin:

```bash
command -v base64 jq
kubectl get secret openfaas-license -n openfaas \
  -o jsonpath='{.data.license}' |
  base64 --decode |
  cut -d. -f2 |
  tr '_-' '/+' |
  awk '{m=length($0)%4; if(m==2)$0=$0"=="; else if(m==3)$0=$0"="; print}' |
  base64 -d | jq '{products, exp}'
```

For either fallback, confirm the expected entitlement in `products` and compare `exp` with the current time. This fallback does not verify the signature or provide the plugin's full status checks; report that limitation. Treat decoded claims as sensitive metadata.

### Check current license status for an installation

Follow the plugin availability and interpretation guidance in [Inspect a cluster license](#inspect-a-cluster-license). Retrieve the static cluster license directly from the existing Secret without requiring a source file or changing the Secret:

```bash
kubectl get secret openfaas-license -n openfaas \
  -o jsonpath='{.data.license}' |
  base64 --decode |
  faas-cli pro license print |
  awk '/^(Expires in|Products|Status):/'
```

### Replace an existing license

Replace a license only when the user explicitly requests it; do not infer permission from a general upgrade request. Confirm the cluster context, existing Secret, and affected Deployments before mutation:

```bash
kubectl config current-context
kubectl get secret openfaas-license -n openfaas
kubectl get deployments -n openfaas
```

[Prepare the supplied replacement file](#prepare-a-supplied-license-file) and [inspect its entitlement and status](#inspect-a-cluster-license). Proceed only if both are suitable for this installation, then follow the [official license update procedure](https://docs.openfaas.com/deployment/pro/#need-to-update-your-license):

```bash
kubectl delete secret openfaas-license -n openfaas
kubectl create secret generic openfaas-license \
  -n openfaas \
  --from-file license="$OPENFAAS_LICENSE_JWT"
kubectl rollout restart deployment -n openfaas
kubectl rollout status deployment -n openfaas --timeout=5m
```

Do not restart workloads unless the replacement Secret was created successfully. Clean the restrictive temporary directory, then follow the authenticated checks in [operations.md](operations.md) and confirm all OpenFaaS Pods are healthy. Do not print the license, decoded payload, subject, or account email during replacement or verification.

## Select a deployment profile

Choose from the user's stated intent rather than mixing development and production settings. Confirm all values against the current [Pro deployment](https://docs.openfaas.com/deployment/pro/) and [values-pro.yaml](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values-pro.yaml).

For a local evaluation or staging cluster, keep single replicas and development dashboard access:

```yaml
openfaasPro: true
clusterRole: true

operator:
  create: true
  leaderElection:
    enabled: false

gateway:
  replicas: 1
  upstreamTimeout: 10m
  writeTimeout: 10m2s
  readTimeout: 10m2s

autoscaler:
  enabled: true

dashboard:
  enabled: true
  publicURL: localhost

queueWorker:
  replicas: 1

queueWorkerPro:
  maxInflight: 50

nats:
  streamReplication: 1

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
```

For production, start from the current `values-pro.yaml` posture: use three gateway and queue-worker replicas with operator leader election, real dashboard/gateway FQDNs, a durable dashboard signing key, and production storage/availability decisions. Retain only the deliberate overrides in the installation's canonical values file.

The gateway Deployment runs the gateway and provider/operator containers together in each gateway Pod. Enabling `operator.create` does not create a separate operator Deployment. Verify the provider/operator container inside the gateway Pods; a missing standalone operator Deployment is not an error.

Keep `operator.leaderElection.enabled: true` whenever more than one gateway replica runs. `clusterRole: true` is required for node-level metrics and CPU autoscaling; For Enterprises also uses it for multiple function namespaces.

The JetStream queue-worker is selected automatically when `openfaasPro: true`; the current chart has no `queueMode` value. The bundled NATS server is a single peer, so keep `nats.streamReplication: 1`. For critical asynchronous workloads, discuss an external, persistent, multi-replica NATS deployment rather than raising this value on bundled NATS.

## Pre-install Secret checklist

Before running Helm, confirm every Secret referenced by the selected values exists:

- `openfaas-license` for every Standard or Enterprise installation
- `dashboard-jwt` only when `dashboard.signingKeySecret: dashboard-jwt` is set
- `issuer-key` in `openfaas`, containing a non-empty `issuer.key` entry, whenever `iam.enabled: true`, including installations without dashboard SSO. Helm does not create this Secret. If absent, [generate the issuer signing key and create the Secret](iam-sso.md#secret-handling) before deployment.
- Additional dashboard/SSO Secrets required by the matrix in [iam-sso.md](iam-sso.md#secret-handling) for the selected configuration

When `iam.enabled: true`, verify the issuer Secret without displaying its contents:

```bash
kubectl get secret issuer-key -n openfaas \
  -o go-template='{{if index .data "issuer.key"}}present{{end}}' |
  grep -qx present
```

Do not run `helm upgrade --install` with IAM enabled until this check succeeds. If the Secret exists but its required entry is missing or empty, resolve that configuration issue before deployment; do not silently overwrite an existing signing key.

## Dashboard signing key

For development, the signing key may be left blank and generated automatically, but sessions become invalid after a restart. For a durable installation, preserve an existing `dashboard-jwt` Secret. Create it only when absent:

```bash
OPENFAAS_KEYS_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_KEYS_DIR"
openssl ecparam -genkey -name prime256v1 -noout -out "$OPENFAAS_KEYS_DIR/key"
openssl ec -in "$OPENFAAS_KEYS_DIR/key" -pubout -out "$OPENFAAS_KEYS_DIR/key.pub"
kubectl create secret generic dashboard-jwt \
  -n openfaas \
  --from-file=key="$OPENFAAS_KEYS_DIR/key" \
  --from-file=key.pub="$OPENFAAS_KEYS_DIR/key.pub"
rm -f "$OPENFAAS_KEYS_DIR/key" "$OPENFAAS_KEYS_DIR/key.pub"
rmdir "$OPENFAAS_KEYS_DIR"
```

If secret creation fails, securely remove the temporary files after diagnosing the failure. Do not overwrite an existing Secret without approval.

Use the current chart-documented loopback URL for port-forward-only access. Use the real FQDN when exposing the dashboard. Read [iam-sso.md](iam-sso.md) before configuring an IAM-protected dashboard; its gateway issuer URL must also be reachable from the dashboard pod, so a user's loopback gateway URL is not suitable for dashboard SSO.

## Enterprise additions

With `clusterRole: true`, add a function namespace when requested:

```bash
kubectl create namespace <function-namespace>
kubectl label namespace <function-namespace> openfaas=1
```

Do not assume that every Enterprise installation needs IAM. IAM can be configured during or after core installation. Function Builder, dedicated JetStream queue-workers, event connectors, external NATS, Grafana dashboards, air-gap mirroring, ingress, and TLS are separate components or workflows.

References:

- [Standard and For Enterprises deployment](https://docs.openfaas.com/deployment/pro/)
- [Production preparation](https://docs.openfaas.com/architecture/production/)
- [Dashboard](https://docs.openfaas.com/openfaas-pro/dashboard/)
- [Air-gap installation](https://docs.openfaas.com/openfaas-pro/airgap/)
