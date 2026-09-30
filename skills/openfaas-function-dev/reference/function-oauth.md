# OAuth/OIDC browser login with of-watchdog

Source: [OpenFaaS function OAuth reference](https://docs.openfaas.com/reference/function-oauth/). Use an image with the OAuth-enabled `of-watchdog`; watchdog settings do nothing in an image that runs the handler directly. The watchdog runs OAuth 2.0 Authorization Code with PKCE, handles callbacks, signs a session JWT in a cookie, and requires a valid cookie before forwarding application requests. It serves the sign-in page itself. The handler only needs to implement authorization when access depends on the signed-in user's identity or permissions. This is browser session login: the code grant flow cannot be completed headlessly, so `curl`, the CLI, and other services cannot invoke the function. Use [IAM function authentication](function-iam-auth.md) for headless callers or when OpenFaaS policies should restrict invocation to authorized callers.

## Configure a function

1. Before building, check that the selected template's `template/<lang>/Dockerfile` (or a custom image's Dockerfile) uses `of-watchdog` 0.12.3 or later. Earlier releases with OAuth support stored the provider's tokens in the session cookie in readable form. For a pre-built image, confirm its watchdog version from its source or runtime logs.
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

Other documented options are `oauth_scopes` (space- or comma-separated; default `openid`, always included for OIDC), `oauth_cookie_name` (default `of_session`), `oauth_login_cookie_name` (default `of_login`, must differ from the session cookie name), `oauth_login_redirect` (default `oauth_base_url`), `oauth_session_default_ttl` (default `1h` when the provider supplies no expiry), `oauth_session_ttl`, `oauth_allow_http` (development only; default `false`), and `oauth_token_auth_method`. Check the [reference](https://docs.openfaas.com/reference/function-oauth/#optional-configuration) before changing session lifetime: `oauth_session_ttl` can extend the cookie beyond provider token expiry; the provider is not contacted again during the session.

## Handle the session

The watchdog redirects application requests without a valid session cookie to its sign-in page at `GET {oauth_base_url}/auth/login`. The page's Sign in button submits `POST {oauth_base_url}/auth/login`, which starts the provider flow. The watchdog handles `GET {oauth_base_url}/auth/callback` and sets the session cookie. Submit `POST {oauth_base_url}/auth/logout` to clear the cookie and return to the sign-in page. Login or callback failures render the watchdog's error page. Do not implement the code exchange or sign-in page in the handler.

On an application request, the watchdog forwards a valid session cookie intact and redirects requests with a missing, invalid, or expired cookie to the sign-in page. The handler does not need to check whether the visitor is signed in.

### Session cookie contents

The cookie is an HS256 JWT signed with the `oauth_signing_key` secret. `iss` and `aud` both equal `oauth_base_url`, and `cookie_name` is the session cookie name. The provider's access and ID tokens are discarded after login and are **never** in the cookie, so a handler cannot call provider APIs on the user's behalf. If that is a requirement, use authentication in the handler instead of the watchdog. The identity claims sit under `value`:

| Provider | `value` |
|----------|---------|
| OIDC (ID token returned) | `sub` (`fed:<provider subject>`), `fed:iss` (provider issuer), and `email` and `name` when present |
| Plain OAuth, e.g. a GitHub OAuth App | `{}`: proves sign-in only, with no identity. Use an OIDC provider when the handler needs to know who the user is. |

Base authorization on `value.sub`, not on the mutable `email` or `name`.

### Read the session in a handler

Only parse the cookie when the handler needs the user's identity. **Always verify it; never just base64-decode the payload.** Template servers listen on all interfaces in the Pod (for example `golang-middleware` on `:8082` and `python3-http` on `0.0.0.0:5000`), so a caller that reaches the Pod directly bypasses the watchdog. Bind the signing-key secret to the function too, and verify:

- the HS256 signature with the base64-decoded key from `/var/openfaas/secrets/<oauth_signing_key>`
- that `iss` and `aud` equal the `oauth_base_url` environment variable, which is also visible to the handler
- that `exp` is present and in the future
- that `cookie_name` matches

Go ([golang-jwt](https://github.com/golang-jwt/jwt)):

```go
type Session struct {
	jwt.RegisteredClaims
	CookieName string `json:"cookie_name"`
	Value      struct {
		Subject string `json:"sub"`
		Issuer  string `json:"fed:iss"`
		Email   string `json:"email"`
		Name    string `json:"name"`
	} `json:"value"`
}

// signingKey: base64-decode /var/openfaas/secrets/<oauth_signing_key> once at start-up.
func readSession(r *http.Request) (*Session, error) {
	cookie, err := r.Cookie("of_session")
	if err != nil {
		return nil, err
	}
	baseURL := os.Getenv("oauth_base_url")
	session := &Session{}
	_, err = jwt.ParseWithClaims(cookie.Value, session,
		func(*jwt.Token) (any, error) { return signingKey, nil },
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(baseURL), jwt.WithAudience(baseURL),
		jwt.WithExpirationRequired())
	if err != nil || session.CookieName != "of_session" {
		return nil, errors.New("invalid session")
	}
	return session, nil
}
```

Python ([PyJWT](https://pyjwt.readthedocs.io/)); with `python3-http`, pass `event.headers.get("Cookie")`:

```python
def read_session(cookie_header):
    cookie = SimpleCookie(cookie_header).get("of_session")
    if cookie is None:
        return None
    try:
        claims = jwt.decode(cookie.value, SIGNING_KEY, algorithms=["HS256"],
                            issuer=BASE_URL, audience=BASE_URL,
                            options={"require": ["exp", "iss", "aud"]})
    except jwt.InvalidTokenError:
        return None
    return claims["value"] if claims.get("cookie_name") == "of_session" else None
```

Use `oauth_cookie_name` in place of `of_session` when it is set. Never log the raw cookie, reflect it into a response, or expose it to browser code. For a SPA, have the frontend call a function-side JSON API that verifies the cookie and returns only the claims it needs.

## Verify the flow

- Confirm the deployed image has the OAuth-enabled watchdog and the function has its signing key and, for a confidential client, client secret bound. `faas-cli describe <fn>` shows environment and secret names; `faas-cli logs <fn>` helps diagnose configuration failures.
- Request the function URL without a session cookie and without following redirects. It should redirect to `{oauth_base_url}/auth/login`; a `GET` there should serve the watchdog's sign-in page. Submitting its Sign in button (`POST /auth/login`) should redirect to the provider's authorization endpoint.
- Confirm the provider's registered callback exactly matches `{oauth_base_url}/auth/callback`, including scheme, host, path, and any local port. HTTPS is required unless `oauth_allow_http` is enabled for local development.
- In a browser, visit the public function URL and sign in. Check that the callback returns to the function, a valid cookie reaches the handler, and logout uses POST and returns to the sign-in page. Confirm that a missing or invalid cookie cannot reach the handler.
- For `local-run`, use a local `oauth_base_url` and register its corresponding local callback URI with the provider. Mount test secret files via the project-root `.secrets/` directory. Run `local-run` in the background or tmux as described in the main skill.
