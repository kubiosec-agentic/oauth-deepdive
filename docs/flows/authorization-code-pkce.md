# 4.1 Authorization Code (+ PKCE)

**Who this is for:** every user-facing app today — web apps, SPAs, mobile, desktop. With PKCE, it's the only flow OAuth 2.1 endorses for human-driven access.

**Why it exists:** the authorization code is a single-use credential that's safe to pass through the user-agent (browser address bar), because exchanging it for an actual access token requires the client's secret (confidential clients) and/or the PKCE verifier (public clients).

## The sequence

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant B as Browser / User-Agent
    participant C as Client
    participant AS as Authorization Server
    participant RS as Resource Server

    C->>C: Generate code_verifier (43–128 random chars)<br/>code_challenge = BASE64URL(SHA256(verifier))
    C->>B: 302 → /authorize?response_type=code<br/>&code_challenge=...&state=...
    B->>AS: GET /authorize
    AS->>U: Login + consent screen
    U->>AS: Authenticate + approve
    AS->>B: 302 → redirect_uri?code=ABC&state=...
    B->>C: GET /cb?code=ABC&state=...
    C->>C: Verify state matches stored value
    C->>AS: POST /token<br/>grant_type=authorization_code<br/>&code=ABC&code_verifier=...
    AS->>AS: Check SHA256(verifier) == challenge
    AS->>C: 200 { access_token, refresh_token, expires_in }
    C->>RS: GET /resource<br/>Authorization: Bearer ...
    RS->>RS: Validate token (sig + aud + exp + scope)
    RS->>C: 200 resource data
```

## The dance, in detail

1. Client generates a `code_verifier` (random 43–128 chars) and derives `code_challenge = BASE64URL(SHA256(code_verifier))`.
2. Client redirects the user to `/authorize` with `response_type=code`, `code_challenge`, `code_challenge_method=S256`.
3. AS authenticates the user, gets consent, redirects back with `?code=…&state=…`.
4. Client POSTs to `/token` with the `code` *and* the original `code_verifier`.
5. AS verifies `SHA256(code_verifier) == code_challenge`, returns access (and optionally refresh) token.

## HTTP — step 2, the authorization request

```http
GET /authorize?
    response_type=code
    &client_id=s6BhdRkqt3
    &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
    &scope=read%3Amail%20write%3Acalendar
    &state=xyzABC123
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
    &resource=https%3A%2F%2Fapi.example.com HTTP/1.1
Host: as.example.com
```

After the user authenticates and consents:

```http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=xyzABC123
```

## HTTP — step 4, the token exchange

```http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
&resource=https%3A%2F%2Fapi.example.com
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type":   "Bearer",
  "expires_in":   3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "scope":        "read:mail write:calendar"
}
```

The `Authorization: Basic` header is only present for confidential clients. Public clients omit it (or use `token_endpoint_auth_method: none`) and lean on PKCE for code-to-token binding.

## Why PKCE matters

Without PKCE, a malicious app on the same device (mobile) or a network-positioned attacker (SPA over a redirect URI with permissive matching) could grab the `code` from the redirect and exchange it for a token. PKCE blocks this: the attacker doesn't know the original `code_verifier`, so step 9 in the diagram fails.

PKCE is now mandatory in OAuth 2.1 **for every client**, not just public ones. Confidential clients gain defense-in-depth at near-zero cost.

## State and CSRF

The `state` parameter is your CSRF token for the browser leg — the client must persist it before redirect and verify it on callback. Skip this and you have a usable account-takeover bug: an attacker initiates an auth flow against their own account, sends the resulting callback URL to a victim, and the victim's client links the attacker's tokens to the victim's session.

## Practical guidance

- Use PKCE with `S256` *always*. `plain` exists in the spec but should not be used.
- Use `state`. Use [`nonce` for OIDC](../08-oidc.md).
- Pin the redirect URI to an exact string — no wildcards, no trailing-slash drift.
- For SPAs, never put refresh tokens in `localStorage`. Use HTTP-only cookies via a backend-for-frontend, or a service worker.
- Validate `iss` on the callback ([RFC 9207](../06-rfc-reference.md)) when your AS supports it — defends against mix-up attacks.

---

← [Flows overview](README.md) · ↑ [README](../../README.md) · → Next: [Implicit (deprecated)](implicit.md)
