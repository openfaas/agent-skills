# Standard and For Enterprises

Both editions use the same chart and `openfaasPro: true`; entitlements come from the cluster license. Standard targets a single team and function namespace. For Enterprises adds features such as multiple function namespaces and IAM/SSO.

## Contents

- [Cluster license](#cluster-license)
- [Recommended minimal values](#recommended-minimal-values)
- [Dashboard signing key](#dashboard-signing-key)
- [Enterprise additions](#enterprise-additions)

## Cluster license

Verify that the license file exists without displaying it. Preserve an existing `openfaas-license` Secret unless the user explicitly requested license replacement.

For an Enterprise or IAM request, inspect only the non-secret entitlement and expiry claims before provisioning. Do not print the JWT or its complete payload:

```bash
jq -R 'split(".")[1] | @base64d | fromjson | {products, exp}' \
  <cluster-license-path>
```

Confirm that `products` contains the expected OpenFaaS entitlement and that `exp` has not passed. Treat even decoded claims as sensitive metadata and report only the product needed for the selected workflow and whether the license is currently valid.

For a new installation:

```bash
kubectl create secret generic openfaas-license \
  -n openfaas \
  --from-file license=<cluster-license-path>
```

For an intentional license update, follow the [official license update procedure](https://docs.openfaas.com/deployment/pro/#need-to-update-your-license). It requires replacing the Secret and restarting deployments; do not infer permission to do this from a general upgrade request.

## Recommended minimal values

Start with the current recommendations from [Pro deployment](https://docs.openfaas.com/deployment/pro/) and [values-pro.yaml](https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/values-pro.yaml), then confirm them against the current chart:

```yaml
openfaasPro: true
clusterRole: true

operator:
  create: true
  leaderElection:
    enabled: true

gateway:
  replicas: 3
  upstreamTimeout: 10m
  writeTimeout: 10m2s
  readTimeout: 10m2s

autoscaler:
  enabled: true

dashboard:
  enabled: true
  publicURL: localhost
  signingKeySecret: dashboard-jwt

queueWorker:
  replicas: 3

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

Keep `operator.leaderElection.enabled: true` whenever more than one gateway replica runs. `clusterRole: true` is required for node-level metrics and CPU autoscaling; For Enterprises also uses it for multiple function namespaces.

The JetStream queue-worker is selected automatically when `openfaasPro: true`; the current chart has no `queueMode` value. The bundled NATS server is a single peer, so keep `nats.streamReplication: 1`. For critical asynchronous workloads, discuss an external, persistent, multi-replica NATS deployment rather than raising this value on bundled NATS.

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
