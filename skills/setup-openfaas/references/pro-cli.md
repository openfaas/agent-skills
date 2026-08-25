# faas-cli and Pro plugin

Install the base CLI for all editions. Basic Auth operation of a Standard or Enterprise gateway does not require the Pro plugin or `LICENSE_CLI`; install and activate the plugin only when the requested workflow uses a plugin feature such as IAM authentication or a Pro build command.

## Install

Prefer the installation methods in the [official CLI installation guide](https://docs.openfaas.com/cli/install/). For example:

```bash
arkade get faas-cli
faas-cli version
```

When a Pro plugin feature is required:

```bash
faas-cli plugin get pro
faas-cli pro --help
```

## Activate the Pro plugin

The CLI license is separate from the Kubernetes cluster license.

- For CI/CD, air-gapped use, or a provided static CLI license, store it at `~/.openfaas/LICENSE_CLI`, or use one of the documented `--license-file` or `OPENFAAS_LICENSE` mechanisms. Do not display the JWT.
- For an eligible GitHub organisation, activate interactively:

  ```bash
  faas-cli pro enable
  ```

Do not copy the cluster `LICENSE` into `LICENSE_CLI` unless OpenFaaS supplied the same credential for both purposes.

For unattended tests that actually require a Pro plugin feature, require a static CLI license through `LICENSE_CLI`, `--license-file`, or injected `OPENFAAS_LICENSE`. Do not start `faas-cli pro enable` because its browser/device flow requires a human. A missing CLI license is not a blocker for a Basic Auth installation.

## Authenticate without IAM

Use the chart's default NodePort when the Kubernetes node is reachable from the current shell (`127.0.0.1` applies only when the node is local):

```bash
export OPENFAAS_URL=http://<node-address>:31112
```

Otherwise use ingress, or run a port-forward in the background with logs and guaranteed cleanup:

```bash
OPENFAAS_PF_LOG=$(mktemp)
kubectl port-forward -n openfaas svc/gateway 8080:8080 >"$OPENFAAS_PF_LOG" 2>&1 &
OPENFAAS_PF_PID=$!
trap 'kill "$OPENFAAS_PF_PID" 2>/dev/null || true; wait "$OPENFAAS_PF_PID" 2>/dev/null || true; rm -f "$OPENFAAS_PF_LOG"' EXIT
export OPENFAAS_URL=http://127.0.0.1:8080
```

Then authenticate without exposing the password:

```bash
OPENFAAS_PASSWORD=$(kubectl get secret -n openfaas basic-auth \
  -o jsonpath='{.data.basic-auth-password}' | base64 --decode)
OPENFAAS_READY=0
for OPENFAAS_ATTEMPT in $(seq 1 60); do
  if printf %s "$OPENFAAS_PASSWORD" | faas-cli login \
      --username admin --password-stdin >/dev/null 2>&1 \
      && faas-cli list >/dev/null 2>&1; then
    OPENFAAS_READY=1
    break
  fi
  sleep 5
done
unset OPENFAAS_PASSWORD
test "$OPENFAAS_READY" -eq 1
unset OPENFAAS_READY OPENFAAS_ATTEMPT
faas-cli version
faas-cli list
```

Do not print the password or place it on the command line. Use this Basic Auth path for CE and for Standard/Enterprise when IAM is disabled. Stop and reap any background port-forward after the last gateway or dashboard check.

## Authenticate with IAM/SSO

After the OIDC `JwtIssuer`, Policies, and Roles are configured, use the Pro plugin instead of Basic Auth:

```bash
faas-cli pro auth \
  --client-id <client-id> \
  --authority https://oidc.example.com \
  --gateway https://gateway.example.com
```

Subsequent refreshes generally need only `faas-cli pro auth --gateway <url>`. For non-interactive client-credentials use, follow the official `--client-secret` guidance; never expose the secret in logs or shell history.

Register the documented loopback callback with the IdP. WSL uses a `localhost` callback variation; consult the documentation rather than guessing.

References:

- [CLI installation and Pro activation](https://docs.openfaas.com/cli/install/)
- [SSO with faas-cli](https://docs.openfaas.com/openfaas-pro/sso/cli/)
