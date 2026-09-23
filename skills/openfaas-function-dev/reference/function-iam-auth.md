# IAM function authentication

Source: [OpenFaaS Function Authentication](https://docs.openfaas.com/openfaas-pro/iam/function-authentication/). Use this OpenFaaS Pro IAM option when the requirement is to restrict **invocation of a function** to authorized users or services. IAM roles and policies grant `Function:Invoke`; the caller supplies a function access token. This is different from [watchdog OAuth/OIDC browser login](function-oauth.md), which creates an application session and forwards even requests without a session cookie. IAM does not provide the function's browser login or its application-specific authorization decisions.

## Enable and authorize

1. Confirm OpenFaaS IAM is enabled and configured and the function image has a watchdog version compatible with function authentication. The [documentation](https://docs.openfaas.com/openfaas-pro/iam/function-authentication/#enable-built-in-authentication) lists the supported classic-watchdog and of-watchdog versions. An image without a watchdog cannot act on `jwt_auth`.
2. Set `jwt_auth` on the function in `stack.yaml`, then use the normal `faas-cli diff` and tagged deploy workflow:

   ```yaml
   functions:
     reports:
       skip_build: true
       image: ghcr.io/acme/reports:1.0.0
       environment:
         jwt_auth: "true"
   ```

3. Ensure an IAM Policy allows the `Function:Invoke` action for the intended function and a Role binds that policy to the intended principal. Policy resources can target `*`, a namespace such as `staging:*`, or one function such as `openfaas-fn:reports`. Prefer the narrow resource that matches the requirement. Function configuration alone does not grant anyone access; follow the [policy and role example](https://docs.openfaas.com/openfaas-pro/iam/function-authentication/#define-roles-and-policies). If IAM is not configured, identify that platform prerequisite rather than changing the OpenFaaS control plane as part of a function edit.

IAM function authentication is opt-in per function. Without `jwt_auth`, functions remain invocable without a function access token by default. Setting `jwt_auth` does not replace the function's own checks for what an authenticated caller may do with application data.

## Invoke and verify

A caller first obtains an OIDC ID token from an identity provider registered with OpenFaaS or an OpenFaaS API access token, then exchanges it at the gateway's `/oauth/token` endpoint for a **function access token**. The optional token-exchange `audience` can limit that token to a function or subset. Send the resulting token as `Authorization: Bearer <function-access-token>` to `/function/<name>`; do not send the initial ID/API token as though it were the function access token. See the [token exchange and curl examples](https://docs.openfaas.com/openfaas-pro/iam/function-authentication/#invoke-authenticated-functions).

For CLI verification, authenticate the CLI with `faas-cli pro auth`, then invoke the function with `faas-cli invoke reports --auth` (or pipe a body into it). The `--auth` flag sends the function access token on the first attempt; without it, the CLI can detect the function's IAM challenge and retry. Check that an authorized principal can invoke and a principal without `Function:Invoke` cannot. If `jwt_auth` appears ineffective, inspect the deployed environment and watchdog version before changing policy.
