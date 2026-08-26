# Local registry for single-node K3s

Use this workflow only when the user explicitly requests the Function Builder with the local, unauthenticated registry profile. This profile is for a single-node K3s evaluation or development cluster. Do not generalize it to multi-node clusters or other Kubernetes providers.

## Contents

- [Confirm the boundary](#confirm-the-boundary)
- [Deploy the registry](#deploy-the-registry)
- [Configure K3s containerd](#configure-k3s-containerd)
- [Verify the registry path](#verify-the-registry-path)

## Confirm the boundary

Confirm all of these before changing the cluster or node:

- exactly one Kubernetes node is registered and it is the K3s server node
- the `openfaas` namespace exists
- the `local-path` StorageClass exists
- the agent has shell and `sudo` access to the K3s server node
- the registry Service will remain `ClusterIP`-only, without authentication, ingress, or a NodePort
- the user accepts a brief K3s interruption; ask immediately before the restart when the cluster is shared or the impact is uncertain

Run Kubernetes commands through the selected kubeconfig. Run `systemctl`, `k3s crictl`, and file operations under `/etc/rancher/k3s` on the K3s server node, using SSH when kubectl is running from another machine. Do not mistake the administrator workstation for the node.

```bash
kubectl get nodes -o wide
kubectl get storageclass local-path
kubectl get namespace openfaas
kubectl get deployment,service,pvc -n openfaas
```

Inspect existing objects before applying the manifest. Preserve an existing registry Deployment, Service, PVC, and their data unless the user explicitly requested replacement.

## Deploy the registry

Apply this manifest from a retained, user-approved configuration location. The Service deliberately remains internal to the cluster:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: openfaas
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: openfaas
spec:
  replicas: 1
  selector:
    matchLabels: {app: registry}
  template:
    metadata:
      labels: {app: registry}
    spec:
      containers:
        - name: registry
          image: registry:3
          ports:
            - {name: registry, containerPort: 5000}
          readinessProbe:
            httpGet: {path: /v2/, port: registry}
          volumeMounts:
            - {name: data, mountPath: /var/lib/registry}
      volumes:
        - name: data
          persistentVolumeClaim: {claimName: registry-data}
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: openfaas
spec:
  type: ClusterIP
  selector: {app: registry}
  ports:
    - {name: registry, port: 5000, targetPort: registry}
```

```bash
kubectl apply -f <registry-manifest>
```

Wait for the registry, then record and validate its Service ClusterIP:

```bash
kubectl rollout status deployment/registry -n openfaas --timeout=2m
REGISTRY_IP=$(kubectl get service registry -n openfaas \
  -o jsonpath='{.spec.clusterIP}')
test -n "$REGISTRY_IP" && test "$REGISTRY_IP" != None
```

The ClusterIP remains stable while the Service is preserved. Re-check it after any Service recreation and update the node mirror if it changed.

## Configure K3s containerd

On the K3s server node, inspect `/etc/rancher/k3s/registries.yaml` without exposing any existing credentials in logs or the final report. Merge the following entry into the existing `mirrors` mapping, substituting the recorded IP:

```yaml
mirrors:
  "registry.openfaas.svc.cluster.local:5000":
    endpoint:
      - "http://REGISTRY_CLUSTER_IP:5000"
```

Do not overwrite the file: it may already contain unrelated mirrors, TLS settings, or credentials. Use a YAML-aware merge when available, preserve ownership and permissions, and validate the resulting YAML before restarting K3s. Keep any backup on the node with permissions at least as restrictive as the source and remove it only after successful verification.

Use the ClusterIP because node-level containerd cannot depend on Kubernetes Service DNS. The explicit `http://` endpoint permits this registry's plaintext v2 API. Do not add a wildcard insecure-registry setting and do not use `localhost:5000`.

After the required restart is approved, run these commands on the K3s server node:

```bash
sudo systemctl restart k3s
sudo k3s kubectl wait --for=condition=Ready node --all --timeout=2m
sudo k3s kubectl rollout status deployment/registry \
  -n openfaas --timeout=2m
```

If K3s does not return Ready, inspect `systemctl status k3s` and recent K3s logs before changing the registry configuration again. Restore only the exact pre-change registry configuration when the merge caused the failure.

## Verify the registry path

Run the HTTP and generated mirror checks on the K3s server node:

```bash
REGISTRY_IP=$(sudo k3s kubectl get service registry -n openfaas \
  -o jsonpath='{.spec.clusterIP}')
test -n "$REGISTRY_IP" && test "$REGISTRY_IP" != None
curl -fsS "http://${REGISTRY_IP}:5000/v2/"
sudo sed -n '1,80p' \
  /var/lib/rancher/k3s/agent/etc/containerd/certs.d/registry.openfaas.svc.cluster.local:5000/hosts.toml
```

Confirm that `hosts.toml` contains the current ClusterIP and an HTTP endpoint. After the Function Builder pushes a known image, prove that node-level containerd can pull it:

```bash
sudo k3s crictl pull \
  registry.openfaas.svc.cluster.local:5000/NAME:TAG
```

Use `registry.openfaas.svc.cluster.local:5000/NAME:TAG` for function images. A successful registry `/v2/` response alone is insufficient: the final verification must cover a builder push, a containerd pull, and a Ready function workload.

References:

- [K3s private registry configuration](https://docs.k3s.io/installation/private-registry)
- [Function Builder](https://docs.openfaas.com/openfaas-pro/builder/)
