# Function Builder

Deploy Function Builder as the separate `pro-builder` Helm release. This workflow includes an unauthenticated, plain-HTTP registry profile only for single-node K3s evaluation or development.

## Confirm scope and entitlement

Confirm the Kubernetes context, install versus upgrade intent, registry design, target platform, retained Builder values path, and payload-secret client path. Preserve existing releases, registry data, values, and Secrets unless replacement is explicit. Never print credentials or expand the task into core OpenFaaS installation or production registry design.

Before mutation, require:

- a healthy authenticated OpenFaaS installation (`faas-cli list` succeeds)
- Builder entitlement in the current cluster license; check the current [Builder chart](https://github.com/openfaas/faas-netes/tree/master/chart/pro-builder) and [documentation](https://docs.openfaas.com/openfaas-pro/builder/) rather than inferring it from `openfaasPro: true`
- an existing normalized `openfaas-license` Secret, preserved without rotation
- capacity for the chart's current BuildKit requests
- `helm`, `kubectl`, `faas-cli`, `curl`, and `base64`

The cluster `LICENSE`, CLI `LICENSE_CLI`, and Builder payload secret are distinct credentials. Inspect the normalized cluster Secret with `faas-cli pro license print`, filtering identity metadata; do not use `license validate`, which checks the separate CLI entitlement:

```bash
OPENFAAS_LICENSE_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_LICENSE_DIR"
trap 'rm -f "$OPENFAAS_LICENSE_DIR/license"; rmdir "$OPENFAAS_LICENSE_DIR" 2>/dev/null || true' EXIT
faas-cli pro license print --help >/dev/null 2>&1 || faas-cli plugin get pro
kubectl get secret openfaas-license -n openfaas \
  -o jsonpath='{.data.license}' | base64 --decode \
  > "$OPENFAAS_LICENSE_DIR/license"
chmod 600 "$OPENFAAS_LICENSE_DIR/license"
faas-cli pro license print "$OPENFAAS_LICENSE_DIR/license" \
  | awk '/^(Products|Status):/'
rm -f "$OPENFAAS_LICENSE_DIR/license"
rmdir "$OPENFAAS_LICENSE_DIR"
trap - EXIT
unset OPENFAAS_LICENSE_DIR
```

```bash
kubectl config current-context
faas-cli list
kubectl get secret openfaas-license -n openfaas
kubectl get node \
  -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory
helm status pro-builder -n openfaas
helm get values pro-builder -n openfaas
kubectl get secret registry-secret payload-secret -n openfaas
```

A missing release is normal for a new install. Treat an existing release as an upgrade and retain its values and Secrets.

## Single-node K3s registry profile

Skip this section when the user supplied a different registry design. For this profile, confirm exactly one K3s server node, `local-path` storage, shell and sudo access to that node, and acceptance of a brief K3s restart. Keep the registry ClusterIP-only and unauthenticated; do not generalize this design to production, multiple nodes, or another Kubernetes provider.

Inspect existing objects and preserve them and their data:

```bash
kubectl get nodes -o wide
kubectl get storageclass local-path
kubectl get deployment,service,pvc -n openfaas
```

Apply this from a retained configuration file:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: registry-data, namespace: openfaas}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources: {requests: {storage: 10Gi}}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: registry, namespace: openfaas}
spec:
  replicas: 1
  selector: {matchLabels: {app: registry}}
  template:
    metadata: {labels: {app: registry}}
    spec:
      containers:
        - name: registry
          image: registry:3
          ports: [{name: registry, containerPort: 5000}]
          readinessProbe: {httpGet: {path: /v2/, port: registry}}
          volumeMounts: [{name: data, mountPath: /var/lib/registry}]
      volumes:
        - name: data
          persistentVolumeClaim: {claimName: registry-data}
---
apiVersion: v1
kind: Service
metadata: {name: registry, namespace: openfaas}
spec:
  type: ClusterIP
  selector: {app: registry}
  ports: [{name: registry, port: 5000, targetPort: registry}]
```

```bash
kubectl apply -f <registry-manifest>
kubectl rollout status deployment/registry -n openfaas --timeout=2m
REGISTRY_IP=$(kubectl get service registry -n openfaas \
  -o jsonpath='{.spec.clusterIP}')
test -n "$REGISTRY_IP" && test "$REGISTRY_IP" != None
```

Before installing Builder, verify from an existing suitable pod that `registry.openfaas.svc.cluster.local:5000` resolves and its `/v2/` endpoint answers over HTTP. Do not expose the Service merely to perform this check.

On the K3s server node, merge this entry into `/etc/rancher/k3s/registries.yaml` with a YAML-aware tool:

```yaml
mirrors:
  "registry.openfaas.svc.cluster.local:5000":
    endpoint:
      - "http://REGISTRY_CLUSTER_IP:5000"
```

Use the Service ClusterIP because node-level containerd cannot depend on Kubernetes Service DNS. Preserve existing mirrors, TLS settings, credentials, ownership, and permissions; validate the merged YAML. Do not use `localhost`, overwrite the file, or add a wildcard insecure-registry rule. Immediately before the restart, reconfirm authorization if the cluster is shared or impact is uncertain.

```bash
sudo systemctl restart k3s
sudo k3s kubectl wait --for=condition=Ready node --all --timeout=2m
sudo k3s kubectl rollout status deployment/registry -n openfaas --timeout=2m
REGISTRY_IP=$(sudo k3s kubectl get service registry -n openfaas \
  -o jsonpath='{.spec.clusterIP}')
test -n "$REGISTRY_IP" && test "$REGISTRY_IP" != None
curl -fsS "http://${REGISTRY_IP}:5000/v2/"
sudo sed -n '1,80p' \
  /var/lib/rancher/k3s/agent/etc/containerd/certs.d/registry.openfaas.svc.cluster.local:5000/hosts.toml
```

Confirm the generated `hosts.toml` contains this ClusterIP and HTTP endpoint. If K3s fails, inspect its status and logs; restore only the exact pre-change registry configuration when the merge caused the failure.

## Secrets and values

Builder requires `registry-secret`, `payload-secret`, and `openfaas-license`. Preserve each existing Secret.

If `payload-secret` exists but its client copy is missing, recover only the client copy. Do not rotate the Secret; rotation invalidates existing build clients.

```bash
umask 077
kubectl get secret payload-secret -n openfaas \
  -o jsonpath='{.data.payload-secret}' | base64 --decode \
  > <payload-secret-client-path>
chmod 600 <payload-secret-client-path>
```

For a new local registry profile where both Builder Secrets are absent, create them and the client copy in one self-contained temporary-file workflow:

```bash
OPENFAAS_BUILDER_SECRET_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_BUILDER_SECRET_DIR"
trap 'rm -f "$OPENFAAS_BUILDER_SECRET_DIR/config.json" "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"; rmdir "$OPENFAAS_BUILDER_SECRET_DIR" 2>/dev/null || true' EXIT
printf '%s\n' '{"auths":{}}' > "$OPENFAAS_BUILDER_SECRET_DIR/config.json"
faas-cli secret generate -o "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
chmod 600 "$OPENFAAS_BUILDER_SECRET_DIR"/*
kubectl create secret generic registry-secret -n openfaas \
  --from-file=config.json="$OPENFAAS_BUILDER_SECRET_DIR/config.json"
kubectl create secret generic payload-secret -n openfaas \
  --from-file=payload-secret="$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
umask 077
install -m 0600 "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt" \
  <payload-secret-client-path>
rm -f "$OPENFAAS_BUILDER_SECRET_DIR/config.json" \
  "$OPENFAAS_BUILDER_SECRET_DIR/payload.txt"
rmdir "$OPENFAAS_BUILDER_SECRET_DIR"
trap - EXIT
unset OPENFAAS_BUILDER_SECRET_DIR
```

If only one Builder Secret exists, preserve it and create only the missing Secret after confirming the partial state; do not run the combined new-install example.

For this local HTTP registry, retain a minimal values file:

```yaml
proBuilder:
  insecureRegistry: true
buildkit:
  rootless: true
```

Rootless mode is the default constraint. Do not silently make BuildKit privileged or reduce requests to force scheduling; diagnose incompatibility and agree on the security or capacity tradeoff first. `insecureRegistry` is allowed only for this bounded profile.

## Install and verify

Render the release and inspect images, resources, security contexts, all three Secret references, and the rendered `insecure: "true"` value before deployment:

```bash
OPENFAAS_BUILDER_RENDERED=$(mktemp)
chmod 600 "$OPENFAAS_BUILDER_RENDERED"
helm template pro-builder openfaas/pro-builder --namespace openfaas \
  -f <builder-values-file> > "$OPENFAAS_BUILDER_RENDERED"
rm -f "$OPENFAAS_BUILDER_RENDERED"
unset OPENFAAS_BUILDER_RENDERED
helm upgrade --install pro-builder openfaas/pro-builder \
  --namespace openfaas -f <builder-values-file>
kubectl rollout status deployment/pro-builder -n openfaas --timeout=5m
```

Unknown Helm values are silently ignored. If the released chart renders `insecure: "false"`, prefer a supporting chart. For this evaluation profile only, a temporary `kubectl set env deployment/pro-builder -n openfaas -c pro-builder insecure=true` compatibility patch is acceptable; report that Helm upgrades revert it. Inspect events and targeted container logs when readiness fails.

Port-forward with a cleanup trap and require the Builder health endpoint within a bounded deadline:

```bash
OPENFAAS_BUILDER_PF_LOG=$(mktemp)
kubectl port-forward -n openfaas deployment/pro-builder 8081:8080 \
  > "$OPENFAAS_BUILDER_PF_LOG" 2>&1 &
OPENFAAS_BUILDER_PF_PID=$!
trap 'kill "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true; wait "$OPENFAAS_BUILDER_PF_PID" 2>/dev/null || true; rm -f "$OPENFAAS_BUILDER_PF_LOG"' EXIT
for OPENFAAS_ATTEMPT in $(seq 1 60); do
  curl -fsS http://127.0.0.1:8081/healthz >/dev/null && break
  sleep 2
done
curl -fsS http://127.0.0.1:8081/healthz >/dev/null
```

For the end-to-end proof, use an existing function when possible. Creating a disposable test and leaving its image in registry storage requires agreement. Explicitly set `OPENFAAS_BUILD_PLATFORM` to `linux/amd64` or `linux/arm64`, based on the user target or the authorized single-node architecture query; never accept the CLI default silently. Generate a fresh architecture-qualified tag for every build:

```bash
case "$OPENFAAS_BUILD_PLATFORM" in linux/amd64|linux/arm64) ;; *) exit 1;; esac
OPENFAAS_BUILD_ARCH=${OPENFAAS_BUILD_PLATFORM#linux/}
OPENFAAS_E2E_TAG="e2e-${OPENFAAS_BUILD_ARCH}-$(date -u +%Y%m%d%H%M%S)"
TAG="$OPENFAAS_E2E_TAG" faas-cli publish \
  --remote-builder http://127.0.0.1:8081 \
  --payload-secret <payload-secret-client-path> \
  --platforms "$OPENFAAS_BUILD_PLATFORM" -f <stack-file>
```

The stack image must be `registry.openfaas.svc.cluster.local:5000/NAME:${TAG}`. After push, require all of these checks:

1. On the K3s node, `sudo k3s crictl pull registry.openfaas.svc.cluster.local:5000/NAME:$OPENFAAS_E2E_TAG` succeeds.
2. An authenticated gateway session still passes `faas-cli list`.
3. Deploy the same fresh tag, wait with a bounded timeout for its exact Deployment, and verify an authenticated invocation response.
4. Remove only an agreed disposable Function and its local files; note that the registry image remains.
5. Stop the port-forward and remove its log.

An HTTP/HTTPS push error calls for checking the rendered/running `insecure` value and exact registry name, not weakening unrelated registries. `ImagePullBackOff` calls for comparing the current ClusterIP with `hosts.toml`. `exec format error` calls for rebuilding the correct platform under another fresh tag, not flushing the node cache.

Report chart/app versions, retained paths, rootless mode and overrides, registry ClusterIP and exposure, mirror generation, authenticated Builder/gateway checks, push, containerd pull, workload readiness, invocation, and any retained test image—never secret values.
