# OpenFaaS Edge

OpenFaaS Edge, also called `faasd-pro`, is the commercial single-host OpenFaaS distribution. It runs as systemd services with containerd and CNI on a dedicated Linux VM or bare-metal host. It is not a Kubernetes distribution and must not be installed with Helm.

This reference covers a basic online installation with the official Edge installer, Basic Auth, and local verification. Read the current [Edge deployment documentation](https://docs.openfaas.com/deployment/edge/) and [installer source](https://github.com/openfaas/faasd/blob/master/hack/install-edge.sh) before changing the host because the installer and supported platforms can change.

## Confirm the design

Use Edge only when the user wants a single-host appliance and accepts that it has no horizontal scaling or high availability. OpenFaaS Standard or For Enterprises on Kubernetes is the appropriate workflow when clustering, Kubernetes integration, or high availability is required.

Confirm all of the following before installation:

- commercial OpenFaaS Edge use and an eligible Edge license file
- a dedicated Linux host with x86_64 or 64-bit Arm architecture
- at least 1 GB RAM recommended, 2-4 vCPU, and 10-25 GB available disk
- local or SSH access with sudo/root privileges and systemd
- no Docker installation and no co-located Kubernetes/containerd/CNI workload
- outbound HTTPS access to the official installer, package repositories, and `ghcr.io`, unless an air-gap workflow was explicitly requested
- whether access remains local on port 8080 or requires a DNS name and trusted TLS

Do not substitute faasd CE for Edge. faasd CE has different licensing and capabilities and is intended for personal, non-commercial use. Do not add the dashboard, TLS, custom services, private-registry configuration, air-gap artifacts, preloaded functions, gVisor, event connectors, or Function Builder to a basic installation unless the user requests them.

## Inspect the host

Run read-only checks before downloading or installing anything:

```bash
uname -m
cat /etc/os-release
command -v systemctl curl
free -h
df -h /
command -v docker || true
systemctl is-active docker 2>/dev/null || true
systemctl is-active containerd 2>/dev/null || true
systemctl is-active faasd 2>/dev/null || true
systemctl is-active faasd-provider 2>/dev/null || true
test ! -e /var/lib/faasd/docker-compose.yaml
```

Stop if Docker is installed, the host is already using containerd or CNI, `/var/lib/faasd/docker-compose.yaml` exists, or either faasd service exists. An existing Edge installation requires an inspection and upgrade plan, not a first install. Do not remove existing runtimes, services, configuration, secrets, or state to make the preflight pass.

Confirm the license exists without displaying it:

```bash
EDGE_LICENSE_SOURCE=<edge-license-path>
test -f "$EDGE_LICENSE_SOURCE"
test -s "$EDGE_LICENSE_SOURCE"
```

Do not print, decode, or copy the license into the working tree. Keep the source path out of shell history when it is itself sensitive.

## Stage the official installer

Download the installer instead of piping it directly into a privileged shell. Use a restrictive temporary directory, inspect the downloaded script, and compare its behavior with the current official documentation:

```bash
EDGE_INSTALLER_DIR=$(mktemp -d)
chmod 700 "$EDGE_INSTALLER_DIR"
curl -sLSf \
  https://raw.githubusercontent.com/openfaas/faasd/refs/heads/master/hack/install-edge.sh \
  -o "$EDGE_INSTALLER_DIR/install-edge.sh"
chmod 700 "$EDGE_INSTALLER_DIR/install-edge.sh"
```

The current installer installs OS packages, installs arkade when absent, downloads the Edge OCI installation bundle, stops any existing faasd/containerd services, and stages the final `faasd install` command in a temporary directory. Review it before execution. If its behavior no longer matches this reference or it would alter unrelated host software, stop and reconcile the current documentation first.

Run the inspected script and retain its output:

```bash
sudo -E "$EDGE_INSTALLER_DIR/install-edge.sh"
```

Do not guess the bundle directory. Follow the exact final installation command printed by this invocation after placing the license. Treat a package-install or download failure as a failed install; do not continue with a partial staging directory.

## Install the license and Edge services

After the installer has staged the bundle, copy the Edge license to the documented location without displaying it:

```bash
sudo install -d -m 0700 /var/lib/faasd/secrets
sudo install -m 0600 "$EDGE_LICENSE_SOURCE" \
  /var/lib/faasd/secrets/openfaas_license
sudo test -s /var/lib/faasd/secrets/openfaas_license
unset EDGE_LICENSE_SOURCE
```

Run the exact `faasd install` command emitted by the installer. It must execute from the staged bundle's `/var/lib/faasd` directory. Do not substitute a path from another run and do not overwrite an existing `/var/lib/faasd/docker-compose.yaml`.

Remove only the temporary installer-download directory after the installation command has completed successfully:

```bash
rm -f "$EDGE_INSTALLER_DIR/install-edge.sh"
rmdir "$EDGE_INSTALLER_DIR"
unset EDGE_INSTALLER_DIR
```

Do not remove the extracted Edge bundle until the installation is verified and its exact path is known to be disposable.

## Verify services and authenticate

Verify systemd and the Edge-managed services before authenticating:

```bash
sudo systemctl is-active containerd
sudo systemctl is-active faasd-provider
sudo systemctl is-active faasd
sudo faasd service list
free -h
df -h /
```

If a service is not active, inspect a bounded log snapshot instead of repeatedly restarting it:

```bash
sudo systemctl --no-pager --full status faasd faasd-provider containerd
sudo journalctl -u faasd -n 100 --no-pager
sudo journalctl -u faasd-provider -n 100 --no-pager
```

Install the base `faas-cli` with a current method from the [official CLI installation guide](https://docs.openfaas.com/cli/install/). The Pro CLI plugin is not required for the basic Edge Basic Auth workflow. From the Edge host, authenticate without printing the generated credentials and require an authenticated API operation to succeed within a bounded deadline:

```bash
export OPENFAAS_URL=http://127.0.0.1:8080
EDGE_USER=$(sudo cat /var/lib/faasd/secrets/basic-auth-user)
EDGE_PASSWORD=$(sudo cat /var/lib/faasd/secrets/basic-auth-password)
EDGE_READY=0
for EDGE_ATTEMPT in $(seq 1 60); do
  if printf %s "$EDGE_PASSWORD" | faas-cli login \
      --username "$EDGE_USER" --password-stdin >/dev/null 2>&1 \
      && faas-cli list >/dev/null 2>&1; then
    EDGE_READY=1
    break
  fi
  sleep 5
done
unset EDGE_PASSWORD
test "$EDGE_READY" -eq 1
unset EDGE_READY EDGE_ATTEMPT
faas-cli version
faas-cli list
unset EDGE_USER
```

Do not deploy a test function for routine installation verification. If the authenticated check misses its deadline, capture service state, recent journals, `sudo faasd service list`, `free -h`, and `df -h /`; do not expose the license or Basic Auth files in diagnostics.

The basic gateway endpoint is plaintext HTTP on localhost port 8080. Do not open it directly to a public network. When remote access is required, stop after local verification and follow the official [Edge TLS guide](https://docs.openfaas.com/edge/tls/) with the requested DNS name and trusted certificate.

## Report the installation

Report:

- host identity, Linux distribution, and architecture
- OpenFaaS Edge/faasd version and installation method
- whether `containerd`, `faasd-provider`, and `faasd` are active
- the non-secret gateway URL and whether it is local-only or protected by TLS
- authenticated `faas-cli version` and `faas-cli list` results
- unresolved production items such as TLS, backups, monitoring, registry access, or air-gap operation

Never report the Edge license, Basic Auth password, or the contents of `/var/lib/faasd/secrets`.

References:

- [OpenFaaS Edge product overview](https://www.openfaas.com/edge/)
- [OpenFaaS Edge deployment](https://docs.openfaas.com/deployment/edge/)
- [OpenFaaS Edge guides](https://docs.openfaas.com/edge/overview/)
- [Edge troubleshooting](https://docs.openfaas.com/edge/troubleshooting/)
- [faasd repository](https://github.com/openfaas/faasd)
