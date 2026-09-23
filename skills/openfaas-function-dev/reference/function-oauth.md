# OAuth/OIDC browser login with of-watchdog

Source: [OpenFaaS function OAuth reference](https://docs.openfaas.com/reference/function-oauth/). Use an image with the OAuth-enabled `of-watchdog`; watchdog settings do nothing in an image that runs the handler directly. The watchdog runs OAuth 2.0 Authorization Code with PKCE, handles callbacks, signs a session JWT in a cookie, and checks that cookie before forwarding requests. The handler remains responsible for deciding which pages or actions require login and which users may access them. This is browser session login for a function, distinct from gateway/IAM function authentication.

## Configure a function

1. Before building, check the selected template's `template/<lang>/Dockerfile` (or the Dockerfile for a custom image) for its `of-watchdog` version. [Version 0.12.0](https://github.com/openfaas/of-watchdog/releases/tag/0.12.0) introduced OAuth/OIDC login; an older watchdog ignores these settings and forwards `/auth/login` to the handler. Update the template's watchdog image or choose a compatible image before building. For a pre-built image, confirm its watchdog version from its source or runtime logs.
2. Register a client with the provider. Set its exact redirect URI to `{oauth_base_url}/auth/callback`, where `oauth_base_url` is the public function URL **including its path**, such as `https://gateway.example.com/function/profile`. Use the URL users actually visit.
3. Choose one provider configuration: for OIDC, set `oauth_issuer_url` and let the watchdog discover endpoints and validate ID tokens; for plain OAuth, set both `oauth_authorization_endpoint` and `oauth_token_endpoint` (for example, a GitHub OAuth App). Do not assume a plain OAuth provider returns an ID token.
4. Create a distinct base64-encoded random 32-byte signing key for each function and store it as an OpenFaaS secret. Store a confidential client's secret as another OpenFaaS secret. Set `oauth_signing_key` and `oauth_client_secret` to the **secret names**, and list those names under the function's `secrets`. Omit `oauth_client_secret` for a public client; PKCE still applies. Never put secret values in `environment` or a committed stack file.

```bash
faas-cli secret generate | faas-cli secret create profile-signing-key --from-file=/dev/stdin
faas-cli secret create profile-client-secret --from-file=client-secret
```

If a secret already exists, use `faas-cli secret update`. Keep local secret files in `.secrets/` next to `stack.yaml`, outside the handler folder, as described in the main skill.

Example OIDC function configuration (replace the example URLs, client ID, and image):

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: https://gateway.example.com
functions:
  profile:
    lang: golang-middleware
    handler: ./profile
    image: ghcr.io/acme/profile:latest
    environment:
      oauth_enabled: "true"
      oauth_base_url: https://gateway.example.com/function/profile
      oauth_client_id: profile
      oauth_issuer_url: https://id.example.com
      oauth_client_secret: profile-client-secret
      oauth_signing_key: profile-signing-key
    secrets:
      - profile-client-secret
      - profile-signing-key
```

For a public client, remove both `oauth_client_secret` and its `secrets` entry. For a provider without OIDC discovery, replace `oauth_issuer_url` with its authorization and token endpoints; set `oauth_scopes` to the provider's needed scopes. GitHub's example uses `oauth_token_auth_method: client_secret_post` and `oauth_scopes: read:user`. The default token authentication method is `client_secret_basic`.

| Setting | Purpose |
|---|---|
| `oauth_enabled` | Set to `"true"`; otherwise login is disabled. |
| `oauth_base_url` | Public function URL, including path; determines callback URL, cookie path, and session issuer/audience. |
| `oauth_client_id` | Provider client ID. |
| `oauth_issuer_url` | OIDC issuer URL; use instead of explicit endpoints. |
| `oauth_authorization_endpoint`, `oauth_token_endpoint` | Both required for plain OAuth without OIDC discovery. |
| `oauth_client_secret` | Name of a bound OpenFaaS secret, when the client has one. |
| `oauth_signing_key` | Name of a bound OpenFaaS secret containing the base64-encoded 32-byte key. |

Other documented options are `oauth_scopes` (space- or comma-separated; default `openid`, always included for OIDC), `oauth_cookie_name` (default `of_session`), `oauth_login_cookie_name` (default `of_login`, must differ from the session cookie name), `oauth_login_redirect` (default `oauth_base_url`), `oauth_logout_redirect` (default `{oauth_base_url}/auth/login`), `oauth_error_redirect`, `oauth_session_default_ttl` (default `1h` when the provider supplies no expiry), `oauth_session_ttl`, `oauth_allow_http` (development only; default `false`), and `oauth_token_auth_method`. Check the [reference](https://docs.openfaas.com/reference/function-oauth/#advanced-configuration) before changing session lifetime: `oauth_session_ttl` can extend the cookie beyond provider token expiry but does not refresh the embedded token.

## Handle the session

The watchdog serves `GET {oauth_base_url}/auth/login`, `GET {oauth_base_url}/auth/callback`, and `POST {oauth_base_url}/auth/logout`. Send unauthenticated visitors to the login URL from a sign-in link or handler redirect; submit a POST to logout. Do not implement the code exchange in the handler.

On an ordinary request, **a request without the session cookie is forwarded unchanged**, so public pages work. A valid session cookie is forwarded intact; an invalid or expired one gets HTTP 401 from the watchdog. To protect a route, the handler must check for the session cookie and redirect to login or return 401/403 as appropriate. The cookie is a signed JWT whose `value` claim contains the provider's `id_token` and/or `access_token`. The watchdog has checked the session JWT's signature, issuer, audience, and expiry before forwarding it. Decode its payload in the handler to read the tokens and make authorization decisions; do this only for requests that pass through the watchdog. An OAuth access token may be opaque, so use the provider's API when claims are unavailable (as in the GitHub example). Avoid logging, reflecting, or exposing the raw cookie or tokens to browser code.

For a SPA, keep profile/token handling in a function-side JSON API and let the frontend call that API with the session cookie, as in the React example.

## Verify the flow

- Confirm the deployed image has the OAuth-enabled watchdog and the function has its signing key and, for a confidential client, client secret bound. `faas-cli describe <fn>` shows environment and secret names; `faas-cli logs <fn>` helps diagnose configuration failures.
- Before attempting a full provider login, request `GET {oauth_base_url}/auth/login` without following redirects. It should return HTTP 302 with a `Location` at the provider's authorization endpoint. If it returns the handler's page or HTTP 200, check the deployed watchdog version first.
- Confirm the provider's registered callback exactly matches `{oauth_base_url}/auth/callback`, including scheme, host, path, and any local port. HTTPS is required unless `oauth_allow_http` is enabled for local development.
- In a browser, visit the public function URL and follow login. Check that the callback returns to the function, a valid cookie reaches the handler, logout uses POST, and a protected route redirects or rejects a visitor without a cookie. An absent cookie alone does **not** cause the watchdog to reject a request.
- For `local-run`, use a local `oauth_base_url` and register its corresponding local callback URI with the provider. Mount test secret files via the project-root `.secrets/` directory. Run `local-run` in the background or tmux as described in the main skill.

The [of-watchdog OAuth examples](https://github.com/welteki/of-watchdog-oauth-examples) provide complete configurations and handlers: `oauth-demo` for OIDC and a Go rendered page, `github-oauth-demo` for explicit OAuth endpoints and GitHub API access, and `react-oauth-demo` for an OIDC login with a React SPA backed by a JSON API. Use the example matching the provider and UI rather than copying provider-specific settings into every function.
