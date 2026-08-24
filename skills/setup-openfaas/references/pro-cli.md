# faas-cli and Pro plugin

Install the base CLI for all editions. Install and activate the Pro plugin for Standard and For Enterprises, including installations that do not enable IAM.

## Install

Prefer the installation methods in the [official CLI installation guide](https://docs.openfaas.com/cli/install/). For example:

```bash
arkade get faas-cli
faas-cli version
```

For Standard or For Enterprises:

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

For unattended E2E tests, require a static CLI license through `LICENSE_CLI`, `--license-file`, or injected `OPENFAAS_LICENSE`. Do not start `faas-cli pro enable` because its browser/device flow requires a human. If no static CLI license is available, complete cluster-side validation and report Pro CLI authentication as a concrete external blocker.

## Authenticate without IAM

Establish the gateway URL through ingress, NodePort, or a foreground port-forward:

```bash
kubectl port-forward -n openfaas svc/gateway 8080:8080
```

In another shell:

```bash
export OPENFAAS_URL=http://127.0.0.1:8080
OPENFAAS_PASSWORD=$(kubectl get secret -n openfaas basic-auth \
  -o jsonpath='{.data.basic-auth-password}' | base64 --decode)
printf %s "$OPENFAAS_PASSWORD" | \
  faas-cli login --username admin --password-stdin
unset OPENFAAS_PASSWORD
faas-cli version
```

Do not print the password or place it on the command line. Use this Basic Auth path for CE and for Standard/Enterprise when IAM is disabled.

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
