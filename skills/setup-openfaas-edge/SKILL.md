---
name: setup-openfaas-edge
description: "Installs and verifies OpenFaaS Edge (faasd-pro) on a dedicated Linux host. Use for commercial single-host Edge deployments, upgrades, and troubleshooting; do not use for OpenFaaS on Kubernetes or faasd Community Edition."
---

# Setup OpenFaaS Edge

Install OpenFaaS Edge as systemd services with its bundled containerd and CNI. It is a commercial single-host appliance, not a Kubernetes or Helm distribution.

## Confirm scope and host safety

Confirm the target host, install versus upgrade intent, eligible Edge license, Linux distribution and architecture, sudo access, sizing, outbound access, and whether the gateway remains local or needs trusted TLS. Use a dedicated x86_64 or 64-bit Arm host. Preserve existing services, state, and secrets; do not add TLS, a dashboard, private-registry support, air-gap assets, event connectors, or Function Builder unless requested.

Use Kubernetes OpenFaaS instead when clustering, horizontal scaling, Kubernetes integration, or high availability is required. Do not substitute faasd Community Edition for commercial Edge.

Before looking for or validating a license, detect runtime conflicts with read-only checks:

```bash
uname -m
cat /etc/os-release
command -v systemctl curl
free -h
df -h /
command -v docker || true
command -v kubelet || true
command -v k3s || true
command -v rke2 || true
command -v k0s || true
systemctl list-unit-files --type=service --no-legend 2>/dev/null \
  | grep -E '^(docker|containerd|kubelet|k3s|k3s-agent|rke2-server|rke2-agent|k0scontroller|k0sworker|faasd|faasd-provider)\.service' \
  || true
for EDGE_UNIT in docker containerd kubelet k3s k3s-agent \
  rke2-server rke2-agent k0scontroller k0sworker faasd faasd-provider
do
  systemctl is-active --quiet "$EDGE_UNIT" 2>/dev/null \
    && printf 'Active runtime unit: %s\n' "$EDGE_UNIT"
done
for EDGE_PATH in /var/lib/faasd/docker-compose.yaml /etc/kubernetes \
  /var/lib/kubelet /etc/rancher/k3s /var/lib/rancher/k3s \
  /etc/rancher/rke2 /var/lib/rancher/rke2 /var/lib/k0s
do
  test ! -e "$EDGE_PATH" || printf 'Existing runtime state: %s\n' "$EDGE_PATH"
done
find /etc/cni/net.d -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null || true
unset EDGE_UNIT EDGE_PATH
```

Stop before license discovery if Docker is installed; containerd or a Kubernetes runtime is installed or active; Kubernetes or non-empty CNI state exists; or faasd state/services exist. `kubectl` alone is not a conflict. Report the exact conflict and require a different dedicated host; never remove or reconfigure it to make preflight pass. An existing Edge installation requires an upgrade plan, not first-install steps.

## Install

Read the current [Edge deployment documentation](https://docs.openfaas.com/deployment/edge/) and [installer source](https://github.com/openfaas/faasd/blob/master/hack/install-edge.sh) before mutation. Confirm the license exists without printing or decoding it:

```bash
EDGE_LICENSE_SOURCE=<edge-license-path>
test -s "$EDGE_LICENSE_SOURCE"
```

Download rather than pipe the installer into a privileged shell:

```bash
EDGE_INSTALLER_DIR=$(mktemp -d)
chmod 700 "$EDGE_INSTALLER_DIR"
curl -sLSf \
  https://raw.githubusercontent.com/openfaas/faasd/refs/heads/master/hack/install-edge.sh \
  -o "$EDGE_INSTALLER_DIR/install-edge.sh"
chmod 700 "$EDGE_INSTALLER_DIR/install-edge.sh"
```

Inspect it and reconcile any difference from current documentation. The installer changes packages and services and stages the final `faasd install` command; stop if it would alter unrelated software. Then run it and retain its output:

```bash
sudo -E "$EDGE_INSTALLER_DIR/install-edge.sh"
```

Do not continue after a partial download or package failure and do not guess the staged bundle path. Place the license only after successful staging:

```bash
sudo install -d -m 0700 /var/lib/faasd/secrets
sudo install -m 0600 "$EDGE_LICENSE_SOURCE" \
  /var/lib/faasd/secrets/openfaas_license
sudo test -s /var/lib/faasd/secrets/openfaas_license
unset EDGE_LICENSE_SOURCE
```

Run the exact final command emitted by the inspected installer from its staged `/var/lib/faasd` directory. Never overwrite `/var/lib/faasd/docker-compose.yaml`. After successful installation, remove only the known installer download:

```bash
rm -f "$EDGE_INSTALLER_DIR/install-edge.sh"
rmdir "$EDGE_INSTALLER_DIR"
unset EDGE_INSTALLER_DIR
```

## Verify and troubleshoot

```bash
sudo systemctl is-active containerd faasd-provider faasd
sudo faasd service list
free -h
df -h /
```

For failures, capture a bounded state snapshot instead of repeatedly restarting:

```bash
sudo systemctl --no-pager --full status faasd faasd-provider containerd
sudo journalctl -u faasd -n 100 --no-pager
sudo journalctl -u faasd-provider -n 100 --no-pager
```

Install the base `faas-cli` using the [official CLI guide](https://docs.openfaas.com/cli/install/); the Pro plugin is unnecessary for Basic Auth. Verify a real authenticated operation within a bounded deadline:

```bash
export OPENFAAS_URL=http://127.0.0.1:8080
EDGE_USER=$(sudo cat /var/lib/faasd/secrets/basic-auth-user)
EDGE_PASSWORD=$(sudo cat /var/lib/faasd/secrets/basic-auth-password)
EDGE_READY=0
for EDGE_ATTEMPT in $(seq 1 60); do
  if printf %s "$EDGE_PASSWORD" | faas-cli login \
      --username "$EDGE_USER" --password-stdin >/dev/null 2>&1 \
      && faas-cli list >/dev/null 2>&1; then EDGE_READY=1; break; fi
  sleep 5
done
unset EDGE_PASSWORD
test "$EDGE_READY" -eq 1
faas-cli version
faas-cli list
unset EDGE_USER EDGE_READY EDGE_ATTEMPT
```

Do not expose the license or Basic Auth files in logs. Do not deploy a test function for routine verification. Plain HTTP on port 8080 must remain local; for remote access, first finish local verification, then follow the requested [Edge TLS workflow](https://docs.openfaas.com/edge/tls/).

After the platform is healthy, use `openfaas-function-dev` for building, configuring, deploying, invoking, or troubleshooting functions.

Report the target, version, installation or upgrade method, service state, non-secret endpoint, authentication result, and unresolved requested work. For deeper faults, use the official [Edge troubleshooting guide](https://docs.openfaas.com/edge/troubleshooting/).
