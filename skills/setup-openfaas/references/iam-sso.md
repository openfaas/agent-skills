# Enterprise IAM and SSO

IAM is an optional OpenFaaS for Enterprises feature. Configure it only after collecting the public gateway URL, OIDC authority, client ID, optional client secret, scopes, dashboard URL, and intended authorization model.

Follow the [IAM overview](https://docs.openfaas.com/openfaas-pro/iam/overview/) and [IAM walkthrough](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/). Confirm current CRDs and Helm values before applying examples.

IAM requires an Enterprise-capable cluster license. Verify its product and expiry claims with the focused procedure in [standard-enterprise.md](standard-enterprise.md) before creating secrets.

## Helm values

Merge this overlay into the complete Pro values from [standard-enterprise.md](standard-enterprise.md); do not deploy it as a standalone values file. Enable the system issuer with the gateway's canonical URL:

```yaml
iam:
  enabled: true
  systemIssuer:
    url: https://gateway.openfaas.example.com
```

For dashboard SSO, configure the IdP and dashboard URL:

```yaml
iam:
  dashboardIssuer:
    url: https://oidc.example.com
    clientId: openfaas
    clientSecret: oauth-client-secret
    scopes:
      - openid
      - profile
      - email

dashboard:
  enabled: true
  publicURL: https://dashboard.openfaas.example.com
  signingKeySecret: dashboard-jwt
```

Leave `clientSecret` blank when the IdP uses PKCE without a secret. If the gateway or IdP uses a private CA, follow the official custom CA bundle instructions.

`iam.systemIssuer.url` must be reachable by CLI users and by the dashboard pod when dashboard SSO is enabled; do not use the operator's workstation loopback URL. For a port-forwarded dashboard, use the current loopback value documented by the chart and register that exact callback with the IdP. For shared or production use, configure real gateway and dashboard FQDNs with trusted TLS before IAM authentication.

## Dashboard callback URL

By default, set `dashboard.publicURL` to the dashboard origin without a path and register `<dashboard.publicURL>/auth/callback`, for example `https://dashboard.openfaas.example.com/auth/callback`.

A dashboard base path is opt-in. Configure one only when the user explicitly requests it and the ingress or proxy serves the same path. Include that path in `dashboard.publicURL` and append `/auth/callback`, for example `https://example.com/openfaas/auth/callback`. Never infer a public base path from the rendered Deployment; derive the callback from `dashboard.publicURL` and verify the redirect URI emitted by the running dashboard before telling the user what to register.

## Secret handling

IAM may require an issuer signing key, dashboard AES key, dashboard signing key, and OAuth client secret. Before creating each Secret:

1. Check whether it already exists.
2. Preserve it unless rotation was explicitly requested.
3. Generate files under a `mktemp -d` directory with mode `0700`.
4. Create the Kubernetes Secret.
5. Remove temporary secret material.

For example, create the issuer key only when absent:

```bash
OPENFAAS_IAM_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_IAM_DIR"
openssl ecparam -genkey -name prime256v1 -noout \
  -out "$OPENFAAS_IAM_DIR/issuer.key"
kubectl create secret generic issuer-key \
  -n openfaas \
  --from-file=issuer.key="$OPENFAAS_IAM_DIR/issuer.key"
rm -f "$OPENFAAS_IAM_DIR/issuer.key"
rmdir "$OPENFAAS_IAM_DIR"
```

Use this requirement matrix:

| Secret | When required | Kubernetes key |
|---|---|---|
| `issuer-key` | IAM system issuer | `issuer.key` |
| `dashboard-jwt` | Durable dashboard sessions; may be omitted for development | `key`, `key.pub` |
| `aes-key` | Dashboard with IAM/SSO | `aes_key` |
| OAuth client Secret named by `iam.dashboardIssuer.clientSecret` | Only when the IdP requires a client secret | `client_secret` |

Generate the dashboard AES Secret only when it is absent:

```bash
OPENFAAS_IAM_DIR=$(mktemp -d)
chmod 700 "$OPENFAAS_IAM_DIR"
openssl rand -hex 16 > "$OPENFAAS_IAM_DIR/aes_key"
chmod 600 "$OPENFAAS_IAM_DIR/aes_key"
kubectl create secret generic aes-key \
  -n openfaas \
  --from-file=aes_key="$OPENFAAS_IAM_DIR/aes_key"
rm -f "$OPENFAAS_IAM_DIR/aes_key"
rmdir "$OPENFAAS_IAM_DIR"
```

Create an OAuth client Secret from a restrictive file supplied through an approved non-command-line channel. Ensure it has no trailing newline, store it with key `client_secret`, and remove the temporary file after creation. Never commit any of these values.

## Identity provider and authorization

Register the CLI callback documented for the chosen IdP and the dashboard callback derived above. Then install or update the required IAM CRDs as directed by the current Pro installation documentation before creating their resources.

Create and verify:

- a `JwtIssuer` for every trusted IdP
- least-privilege `Policy` resources
- `Role` resources binding policies to JWT identities or claims

At least one matching Role is required before SSO users can gain access. Do not deploy a blanket wildcard policy unless the user explicitly requests and understands cluster-wide administration.

After configuration, authenticate and test with the IAM path in [pro-cli.md](pro-cli.md). Test both an allowed operation and a representative denied operation when practical.

CI/CD Web Identity Federation is a separate authorization workflow. Configure it only when requested and follow the provider-specific official guidance.

References:

- [IAM overview](https://docs.openfaas.com/openfaas-pro/iam/overview/)
- [SSO overview and callback URLs](https://docs.openfaas.com/openfaas-pro/sso/overview/)
- [SSO with faas-cli](https://docs.openfaas.com/openfaas-pro/sso/cli/)
- [Dashboard SSO](https://docs.openfaas.com/openfaas-pro/dashboard/)
- [IAM walkthrough](https://www.openfaas.com/blog/walkthrough-iam-for-openfaas/)
