# OAuth/OIDC browser login with of-watchdog

Source: [OpenFaaS function OAuth reference](https://docs.openfaas.com/reference/function-oauth/). Use an image with the OAuth-enabled `of-watchdog`; watchdog settings do nothing in an image that runs the handler directly. The watchdog runs OAuth 2.0 Authorization Code with PKCE, handles callbacks, signs a session JWT in a cookie, and requires a valid cookie before forwarding application requests. It serves the sign-in page itself. The handler only needs to implement authorization when access depends on the signed-in user's identity or permissions. This is browser session login: the code grant flow cannot be completed headlessly, so `curl`, the CLI, and other services cannot invoke the function. Use [IAM function authentication](function-iam-auth.md) for headless callers or when OpenFaaS policies should restrict invocation to authorized callers.

## Configure a function

1. Before building, check that the selected template's `template/<lang>/Dockerfile` (or a custom image's Dockerfile) uses `of-watchdog` 0.12.5 or later. Version 0.12.4 could promote identity claims from an unverified ID token in explicit plain-OAuth endpoint mode, 0.12.3 used an intermediate nested session format, and earlier releases with OAuth support stored the provider's tokens in the session cookie in readable form. For a pre-built image, confirm its watchdog version from its source or runtime logs.
2. Register a client with the provider. Set its exact redirect URI to `{oauth_base_url}/auth/callback`, where `oauth_base_url` is the public function URL **including its path**, such as `https://gateway.example.com/function/profile`. Use the URL users actually visit.
3. Choose one provider configuration: for OIDC, set `oauth_issuer_url` and let the watchdog discover endpoints and validate ID tokens; for plain OAuth, set both `oauth_authorization_endpoint` and `oauth_token_endpoint` (for example, a GitHub OAuth App). Plain OAuth is sign-in-only; the watchdog ignores any unexpected `id_token` because it has not passed OIDC verification.
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

Other documented options are `oauth_scopes` (space- or comma-separated; default `openid`, always included for OIDC), `oauth_groups` (comma-separated exact group names), `oauth_cookie_name` (default `of_session`), `oauth_login_cookie_name` (default `of_login`, must differ from the session cookie name), `oauth_login_redirect` (default `oauth_base_url`), `oauth_session_default_ttl` (default `1h` when the provider supplies no expiry), `oauth_session_ttl`, `oauth_allow_http` (development only; default `false`), and `oauth_token_auth_method`. Check the [reference](https://docs.openfaas.com/reference/function-oauth/#optional-configuration) before changing session lifetime: `oauth_session_ttl` can extend the cookie beyond provider token expiry; the provider is not contacted again during the session.

## Handle the session

The watchdog redirects application requests without a valid session cookie to its sign-in page at `GET {oauth_base_url}/auth/login`. The page's Sign in button submits `POST {oauth_base_url}/auth/login`, which starts the provider flow. The watchdog handles `GET {oauth_base_url}/auth/callback` and sets the session cookie. Submit `POST {oauth_base_url}/auth/logout` to clear the cookie and return to the sign-in page. Login or callback failures render the watchdog's error page. Do not implement the code exchange or sign-in page in the handler.

On an application request, the watchdog forwards a valid session cookie intact and redirects requests with a missing, invalid, or expired cookie to the sign-in page. The handler does not need to check whether the visitor is signed in.

### Session cookie contents

The cookie is a flat HS256 JWT signed with the `oauth_signing_key` secret. `iss` and the single `aud` both equal `oauth_base_url`, and `typ` is `session`. The cookie name is an HTTP transport detail and is not a JWT claim. The provider's access and ID tokens are discarded after login and are **never** in the cookie, so a handler cannot call provider APIs on the user's behalf. If that is a requirement, use authentication in the handler instead of the watchdog.

| Provider | Session identity claims |
|----------|-------------------------|
| OIDC (ID token verified) | `sub` (`fed:<provider subject>`), `fed:iss`, `fed:email`, `fed:email_verified`, `fed:name`, and `fed:groups` when present |
| Plain OAuth, e.g. a GitHub OAuth App | No identity claims: proves sign-in only. Unexpected ID-token data is ignored. Configure `oauth_issuer_url` when the handler needs to know who the user is. |

Base authorization on `sub`, not on the mutable email or name. Only use `fed:email` when `fed:email_verified` is `true` and the provider's semantics are suitable.

By default, all string values in the provider's `groups` claim are copied in provider order. Non-string entries are omitted; if the claim is not an array or contains a non-string entry, the watchdog sets `fed:groups_truncated: true`. If the handler only needs specific groups, set `oauth_groups` to a comma-separated list such as `admins, staff`. The names are exact and case-sensitive; surrounding whitespace, empty entries, and duplicate configuration entries are ignored. Filtering happens before cookie sizing, so required groups are retained even when they were near the end of a large provider claim.

If selected groups still make the JWT too large, the watchdog drops groups from the tail, sets the same truncation marker, and logs counts without names. The marker therefore always means the group information is incomplete. Presence in `fed:groups` can support an allow decision. When the marker is true, treat groups as unknown and deny the request if the decision depends on a group that is absent.

### Read the session in a handler

Only parse the cookie when the handler needs the user's identity. **Always verify it; never just base64-decode the payload.** Template servers listen on all interfaces in the Pod (for example `golang-middleware` on `:8082` and `python3-http` on `0.0.0.0:5000`), so a caller that reaches the Pod directly bypasses the watchdog. Bind the signing-key secret to the function too, and verify:

- the HS256 signature with the base64-decoded key from `/var/openfaas/secrets/<oauth_signing_key>`
- that `iss` and the single `aud` equal the `oauth_base_url` environment variable, which is also visible to the handler
- that `exp` and `iat` are present and valid
- that `typ` equals `session`
- that exactly one non-empty session cookie is present

Go ([golang-jwt](https://github.com/golang-jwt/jwt)):

```go
type Session struct {
	jwt.RegisteredClaims
	Type            string   `json:"typ"`
	FederatedIssuer string   `json:"fed:iss,omitempty"`
	FederatedEmail  string   `json:"fed:email,omitempty"`
	EmailVerified   *bool    `json:"fed:email_verified,omitempty"`
	FederatedName   string   `json:"fed:name,omitempty"`
	FederatedGroups []string `json:"fed:groups,omitempty"`
	GroupsTruncated bool     `json:"fed:groups_truncated,omitempty"`
}

// signingKey: base64-decode /var/openfaas/secrets/<oauth_signing_key> once at start-up.
func readSession(r *http.Request) (*Session, error) {
	cookies := r.CookiesNamed("of_session")
	if len(cookies) != 1 || cookies[0].Value == "" {
		return nil, errors.New("missing or duplicate session cookie")
	}
	baseURL := os.Getenv("oauth_base_url")
	session := &Session{}
	_, err := jwt.ParseWithClaims(cookies[0].Value, session,
		func(*jwt.Token) (any, error) { return signingKey, nil },
		jwt.WithValidMethods([]string{"HS256"}),
		jwt.WithIssuer(baseURL), jwt.WithAudience(baseURL),
		jwt.WithExpirationRequired(), jwt.WithIssuedAt(),
		jwt.WithStrictDecoding())
	if err != nil || session.IssuedAt == nil || session.Type != "session" ||
		len(session.Audience) != 1 || session.Audience[0] != baseURL {
		return nil, errors.New("invalid session")
	}
	return session, nil
}
```

Python ([PyJWT](https://pyjwt.readthedocs.io/)); with `python3-http`, pass `event.headers.get("Cookie")`:

```python
def read_session(cookie_header):
    values = []
    try:
        for part in (cookie_header or "").split(";"):
            parsed = SimpleCookie()
            parsed.load(part)
            cookie = parsed.get("of_session")
            if cookie is not None:
                values.append(cookie.value)
    except CookieError:
        return None
    if len(values) != 1 or not values[0]:
        return None
    try:
        claims = jwt.decode(values[0], SIGNING_KEY, algorithms=["HS256"],
                            issuer=BASE_URL, audience=BASE_URL,
                            options={"require": ["exp", "iat", "iss", "aud", "typ"]})
    except jwt.InvalidTokenError:
        return None
    if claims.get("typ") != "session" or claims.get("aud") != [BASE_URL]:
        return None
    return claims
```

The Python example requires `from http.cookies import CookieError, SimpleCookie`. Use `oauth_cookie_name` in place of `of_session` when it is set. Never log the raw cookie, reflect it into a response, or expose it to browser code. For a SPA, have the frontend call a function-side JSON API that verifies the cookie and returns only the claims it needs.

## Verify the flow

- Confirm the deployed image has the OAuth-enabled watchdog and the function has its signing key and, for a confidential client, client secret bound. `faas-cli describe <fn>` shows environment and secret names; `faas-cli logs <fn>` helps diagnose configuration failures.
- Request the function URL without a session cookie and without following redirects. It should redirect to `{oauth_base_url}/auth/login`; a `GET` there should serve the watchdog's sign-in page. Submitting its Sign in button (`POST /auth/login`) should redirect to the provider's authorization endpoint.
- Confirm the provider's registered callback exactly matches `{oauth_base_url}/auth/callback`, including scheme, host, path, and any local port. HTTPS is required unless `oauth_allow_http` is enabled for local development.
- In a browser, visit the public function URL and sign in. Check that the callback returns to the function, a valid cookie reaches the handler, and logout uses POST and returns to the sign-in page. Confirm that a missing or invalid cookie cannot reach the handler.
- When upgrading from of-watchdog 0.12.4, rotate `oauth_signing_key` if explicit endpoints returned an ID token, invalidating sessions that may contain unverified identity. Version 0.12.3 used a different session shape, and 0.12.2 and earlier stored provider tokens; rotate the signing key when upgrading from those versions too.
- For `local-run`, use a local `oauth_base_url` and register its corresponding local callback URI with the provider. Mount test secret files via the project-root `.secrets/` directory. Run `local-run` in the background or tmux as described in the main skill.
