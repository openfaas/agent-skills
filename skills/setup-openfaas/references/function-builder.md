# Function Builder on single-node K3s

Install the Function Builder only when the user explicitly requests it. This workflow uses an unauthenticated, plain-HTTP registry inside a single-node K3s cluster; it is not a production registry pattern.

## Contents

- [Confirm prerequisites](#confirm-prerequisites)
- [Prepare the registry](#prepare-the-registry)
- [Prepare Builder secrets and values](#prepare-builder-secrets-and-values)
- [Install the Builder](#install-the-builder)
- [Verify an end-to-end build](#verify-an-end-to-end-build)
- [Report and operate](#report-and-operate)

## Confirm prerequisites

Read [local-k3s-registry.md](local-k3s-registry.md) as well as this file. Confirm:

- the core OpenFaaS installation is healthy
- the current official Builder documentation and chart support the selected OpenFaaS license; the chart README currently requires an OpenFaaS for Enterprises license, so do not infer entitlement from `openfaasPro: true` alone
- the normalized `openfaas-license` Secret already exists in `openfaas`; preserve it rather than recreating or rotating it
- `helm`, `kubectl`, `faas-cli`, `base64`, and `curl` are installed
- the single K3s node has capacity for the current Builder requests; the chart currently requests 2 CPU and 2 GiB for BuildKit before other workloads
- one canonical values-file path is selected for the separate `pro-builder` Helm release
- one restrictive client path is selected for the HMAC payload secret used by `faas-cli`

Check current state before mutation:

```bash
kubectl get secret openfaas-license -n openfaas
kubectl get node \
  -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory
helm status pro-builder -n openfaas
helm get values pro-builder -n openfaas
kubectl get secret registry-secret payload-secret -n openfaas
```

A missing Helm release is expected for a new installation. Do not print Secret data. If the Builder already exists, treat the request as an upgrade: retain its values and Secrets unless the user explicitly requested a change or rotation.

## Prepare the registry

Follow [local-k3s-registry.md](local-k3s-registry.md) completely. Finish these checks before installing the Builder:

1. `deployment/registry` is Available.
2. `registry.openfaas.svc.cluster.local:5000` resolves from pods in the cluster.
3. The registry responds to `/v2/` over HTTP.
4. K3s containerd has generated the expected mirror using the current Service ClusterIP.

Keep the registry Service internal. Do not add authentication, TLS, ingress, or provider-specific configuration to this bounded profile.

## Prepare Builder secrets and values

The Builder chart requires `registry-secret`, `payload-secret`, and `openfaas-license`. Preserve any existing Secret. For a new local registry setup, create an empty Docker authentication file and an HMAC payload secret in a restrictive temporary directory:

```bash
OPENFAAS_BUILDER_SECRET_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_BUILDER_SECRET_DIR"
trap 'rm -f "$OPENFAAS_BUILDER_SECRET_DIR/config.json" "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"; rmdir "$OPENFAAS_BUILDER_SECRET_DIR" 2>/dev/null || true' EXIT
printf '%s\n' '{"auths":{}}' > "$OPENFAAS_BUILDER_SECRET_DIR/config.json"
faas-cli secret generate \
  -o "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
chmod 600 "$OPENFAAS_BUILDER_SECRET_DIR/config.json" \
  "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
kubectl create secret generic registry-secret -n openfaas \
  --from-file=config.json="$OPENFAAS_BUILDER_SECRET_DIR/config.json"
kubectl create secret generic payload-secret -n openfaas \
  --from-file=payload-secret="$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
```

Copy the new payload secret to the selected restrictive client path before cleaning the temporary directory, or retrieve an existing payload secret to that path without printing it:

```bash
umask 077
kubectl get secret payload-secret -n openfaas \
  -o jsonpath='{.data.payload-secret}' \
  | base64 --decode > <payload-secret-client-path>
chmod 600 <payload-secret-client-path>
```

Remove the temporary files and unset the variable after both Kubernetes Secrets and the client file are confirmed. Never replace `payload-secret` merely because its client copy is missing; retrieve it instead. Rotation invalidates existing build clients and requires an explicit request.

```bash
rm -f "$OPENFAAS_BUILDER_SECRET_DIR/config.json" \
  "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
rmdir "$OPENFAAS_BUILDER_SECRET_DIR"
trap - EXIT
unset OPENFAAS_BUILDER_SECRET_DIR
```

Create a minimal, retained values file for this separate Helm release:

```yaml
buildkit:
  rootless: true
```

Keep rootless mode unless a diagnosed kernel or runtime incompatibility prevents it. Do not silently fall back to a privileged BuildKit container. Discuss the security impact and node isolation before setting `buildkit.rootless: false`. If the single node cannot satisfy the default requests, agree on intentional Builder resource overrides rather than reducing them merely to make scheduling succeed.

## Install the Builder

Confirm current chart values and images against the official chart immediately before deployment. Render and inspect the separate release:

```bash
OPENFAAS_BUILDER_RENDERED=$(mktemp)
chmod 600 "$OPENFAAS_BUILDER_RENDERED"
helm template pro-builder openfaas/pro-builder \
  --namespace openfaas \
  -f <builder-values-file> > "$OPENFAAS_BUILDER_RENDERED"
```

Inspect targeted Deployment fields, container images, security contexts, resources, and references to `registry-secret`, `payload-secret`, and `openfaas-license`. Do not retain rendered output unnecessarily. Then install or upgrade:

```bash
rm -f "$OPENFAAS_BUILDER_RENDERED"
unset OPENFAAS_BUILDER_RENDERED
helm upgrade --install pro-builder openfaas/pro-builder \
  --namespace openfaas \
  -f <builder-values-file>
kubectl rollout status deployment/pro-builder \
  -n openfaas --timeout=5m
```

Verify both containers and inspect targeted logs only when readiness fails:

```bash
helm status pro-builder -n openfaas
kubectl get deployment/pro-builder service/pro-builder -n openfaas
kubectl get pods -n openfaas -l app=pro-builder
kubectl logs -n openfaas deployment/pro-builder -c pro-builder
kubectl logs -n openfaas deployment/pro-builder -c buildkit
kubectl get events -n openfaas --sort-by=.lastTimestamp
```

## Verify an end-to-end build

Use an existing user-selected function when possible. Otherwise obtain agreement to create a disposable smoke-test function because its image remains in the local registry after the Kubernetes workload is removed.

Manage a local port-forward with a PID and cleanup trap rather than leaving it in the background:

```bash
OPENFAAS_BUILDER_PF_LOG=$(mktemp)
kubectl port-forward -n openfaas deployment/pro-builder \
  8081:8080 > "$OPENFAAS_BUILDER_PF_LOG" 2>&1 &
OPENFAAS_BUILDER_PF_PID=$!
trap 'kill "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true; wait "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true; rm -f "$OPENFAAS_BUILDER_PF_LOG"' EXIT
```

Wait with a bounded retry for `http://127.0.0.1:8081/healthz`:

```bash
OPENFAAS_BUILDER_READY=0
for OPENFAAS_ATTEMPT in $(seq 1 60); do
  if curl -fsS http://127.0.0.1:8081/healthz >/dev/null; then
    OPENFAAS_BUILDER_READY=1
    break
  fi
  sleep 2
done
test "$OPENFAAS_BUILDER_READY" -eq 1
```

Then publish with the remote Builder and the local registry image prefix:

```bash
faas-cli publish \
  --remote-builder http://127.0.0.1:8081 \
  --payload-secret <payload-secret-client-path> \
  -f <stack-file>
```

The function image in the stack file must use:

```yaml
image: registry.openfaas.svc.cluster.local:5000/NAME:TAG
```

After the push succeeds:

1. Run `sudo k3s crictl pull registry.openfaas.svc.cluster.local:5000/NAME:TAG` on the K3s server node.
2. Deploy the function with the authentication method selected in [pro-cli.md](pro-cli.md).
3. Wait for its exact Deployment to become Available with a bounded timeout.
4. Invoke it through the gateway and verify the expected response.
5. Remove only a disposable smoke-test Function and its local build files; report that its image remains in registry storage.
6. Stop the port-forward and remove its log.

```bash
kill "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true
wait "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true
rm -f "$OPENFAAS_BUILDER_PF_LOG"
trap - EXIT
unset OPENFAAS_BUILDER_PF_LOG OPENFAAS_BUILDER_PF_PID \
  OPENFAAS_BUILDER_READY OPENFAAS_ATTEMPT
```

If the push fails with an HTTPS/HTTP mismatch, confirm the image uses the exact registry Service name and port, the Builder version is current, and the registry responds over HTTP from the Builder pod. Do not weaken TLS settings for unrelated registries. If the workload reports `ImagePullBackOff`, compare the current Service ClusterIP with the K3s-generated `hosts.toml` before changing the function.

## Report and operate

Report:

- the `pro-builder` chart/app versions and exact Helm command
- the Builder values-file and payload-secret client paths, without Secret values
- rootless/privileged mode and intentional resource overrides
- the registry Service name and ClusterIP, noting that it is unauthenticated HTTP and ClusterIP-only
- the K3s mirror verification, Builder push, containerd pull, workload readiness, and invocation results
- any retained smoke-test image

For upgrades, inspect both Helm releases, retain the Builder values file, preserve all three Secrets, re-render the chart, and repeat the end-to-end checks. Do not rotate the payload or license Secret as part of an ordinary upgrade.

References:

- [Function Builder API](https://docs.openfaas.com/openfaas-pro/builder/)
- [Function Builder Helm chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder)
