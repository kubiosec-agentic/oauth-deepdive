# 4.1 Authorization Code (+ PKCE)

**Who this is for:** every user-facing app today — web apps, SPAs, mobile, desktop. With PKCE, it's the only flow OAuth 2.1 endorses for human-driven access.

**Why it exists:** the authorization code is a single-use credential that's safe to pass through the user-agent (browser address bar), because exchanging it for an actual access token requires the client's secret (confidential clients) and/or the PKCE verifier (public clients).

---

## What PKCE is, in plain terms

PKCE (pronounced *"pixy"*) stands for **Proof Key for Code Exchange**, defined in **RFC 7636**. You can think of it as a small puzzle the client gives the Authorization Server to solve later, so that even if someone intercepts the authorization code in the middle of the flow, they can't actually redeem it.

The mental model: **the client invents a one-time password before starting the flow, sends a sealed envelope of it to the AS up front, and reveals the original password only when redeeming the code.** Anyone who steals the code from the browser still doesn't have the original password — so the stolen code is useless.

PKCE was originally designed in 2015 (RFC 7636) for mobile apps, but OAuth 2.1 makes it mandatory for **all** clients using authorization code, public and confidential alike. It's two extra parameters and one extra hash computation — near-zero cost for meaningful defence in depth.

## The attack PKCE prevents

Without PKCE, the authorization code is just a string travelling through the browser. If anything between the AS and the client sees it, it's usable. Three concrete attack paths:

```mermaid
flowchart TB
    subgraph PreAttack[Without PKCE]
        U1[User] --> B1[Browser]
        B1 -->|callback URL with code| C1[Real Client]
        Mal[Malicious app<br/>on same device] -.->|intercepts redirect<br/>via custom URL scheme| C1
        Net[Network attacker<br/>or shared subdomain] -.->|steals code from<br/>Referer / history / log| C1
        Net -.->|redeems code directly<br/>at /token| AS1[Authorization Server]
        AS1 -.->|issues token<br/>to attacker| Net
    end
```

Three real ways the code leaks:

1. **Mobile URL-scheme hijacking.** A malicious app on the same device registers itself as a handler for the legitimate app's custom URL scheme (`myapp://`). When the AS redirects to `myapp://cb?code=...`, the OS routes the redirect to whichever app responds. Pre-PKCE, that handed the attacker a redeemable code.
2. **Cross-app interception via permissive redirect URIs.** A client registers `https://example.com/cb` but the AS allows wildcard matching, so `https://attacker.example.com/cb` also works. Attacker phishes the user to start the flow with the attacker's redirect URI, and they get the code.
3. **Browser-history / Referer / log leakage.** The code lives briefly in the URL bar, browser history, server access logs, and `Referer` headers on outbound requests from the callback page. Any of these can leak it to systems that shouldn't have it.

In all three cases, the consequence is the same: an attacker has a fresh authorization code and can exchange it for a real access token by hitting `/token`. The token then lets them act as the user.

## How PKCE blocks it

PKCE adds one secret that **never leaves the legitimate client**. Without that secret, the stolen code is unredeemable.

```mermaid
flowchart TB
    subgraph WithPKCE[With PKCE]
        C2[Real Client] -->|"1- generate code_verifier<br/>(random secret, kept locally)"| C2
        C2 -->|"2- send code_challenge = SHA256(verifier)<br/>at /authorize"| AS2[Authorization Server]
        AS2 -->|3- store challenge with the code it issues| AS2
        AS2 -->|"4- code via redirect"| B2[Browser]
        B2 -->|callback| C2
        C2 -->|"5- redeem: send code + ORIGINAL verifier"| AS2
        AS2 -->|"6- check: SHA256(received_verifier) == stored_challenge ?"| AS2
        AS2 -->|"yes → token; no → reject"| C2
        Att[Attacker who stole<br/>just the code] -.->|7- tries to redeem<br/>without the verifier| AS2
        AS2 -.->|"rejected: no verifier"| Att
    end
```

The attacker can intercept the code as before. But to exchange it for a token at `/token`, they would also need the **`code_verifier`** — the original random secret that the legitimate client generated and kept in its own memory. The verifier never travelled through the browser, never appeared in a URL, never hit a log. The attacker has no way to know it.

PKCE binds the code to the client that started the flow.

## The crypto, in plain terms

PKCE uses three simple ingredients:

```mermaid
flowchart LR
    R["code_verifier<br/>random 43–128 chars<br/>kept secret on the client"] -->|SHA-256 + base64url| C["code_challenge<br/>sent in /authorize<br/>and stored by the AS"]
    R -.->|sent ONLY at /token,<br/>much later| V["AS verifies<br/>SHA-256(verifier) ==<br/>stored challenge"]
```

- **`code_verifier`** — a fresh random string the client generates for *each* authorization flow. 43 to 128 characters, URL-safe. Lives only in the client's memory.
- **`code_challenge`** — `BASE64URL(SHA-256(code_verifier))`. The client sends this to the AS at `/authorize`. The AS stores it alongside the issued authorization code.
- **`code_challenge_method`** — almost always `S256` (SHA-256). The spec also allows `plain` (where challenge == verifier), but `plain` is never the right answer; OAuth 2.1 forbids it.

The whole thing relies on a property of cryptographic hashes: **given the hash (`code_challenge`), you cannot reverse it to find the input (`code_verifier`)**. So even though the attacker sees the challenge, they cannot compute a matching verifier. Only the client that *generated* the verifier can present it later.

When the client redeems the code at `/token`, it sends the original verifier. The AS hashes it on the spot, compares it to the stored challenge, and either accepts or rejects.

---

## The sequence

Now that PKCE is anchored, here's the full Authorization Code + PKCE flow.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant B as Browser / User-Agent
    participant C as Client
    participant AS as Authorization Server
    participant RS as Resource Server

    C->>C: Generate code_verifier (random 43–128 chars)<br/>code_challenge = BASE64URL(SHA256(verifier))
    C->>B: 302 → /authorize?response_type=code<br/>&code_challenge=...&state=...
    B->>AS: GET /authorize
    AS->>U: Login + consent screen
    U->>AS: Authenticate + approve
    AS->>B: 302 → redirect_uri?code=ABC&state=...
    B->>C: GET /cb?code=ABC&state=...
    C->>C: Verify state matches stored value
    C->>AS: POST /token<br/>grant_type=authorization_code<br/>&code=ABC&code_verifier=...
    AS->>AS: Check SHA256(verifier) == stored challenge
    AS->>C: 200 { access_token, refresh_token, expires_in }
    C->>RS: GET /resource<br/>Authorization: Bearer ...
    RS->>RS: Validate token (sig + aud + exp + scope)
    RS->>C: 200 resource data
```

## The dance, in detail

1. Client generates a `code_verifier` (random 43–128 chars) and derives `code_challenge = BASE64URL(SHA256(code_verifier))`.
2. Client redirects the user to `/authorize` with `response_type=code`, `code_challenge`, and `code_challenge_method=S256`.
3. AS authenticates the user, gets consent, and redirects back with `?code=…&state=…`. The AS internally remembers *"code ABC has challenge XYZ"*.
4. Client POSTs to `/token` with the `code` **and** the original `code_verifier`.
5. AS computes `SHA-256(verifier)`, compares to the stored challenge, returns the access (and optionally refresh) token if they match.

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

Notice: the code is in the URL. **That's fine** because PKCE makes the code alone insufficient.

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

The `Authorization: Basic` header is only present for confidential clients. Public clients omit it (or use `token_endpoint_auth_method: none`) and lean entirely on PKCE for code-to-token binding.

## State and CSRF (related but different from PKCE)

PKCE protects the code itself from being usable by an interceptor. The `state` parameter solves a *different* problem: it's a per-request CSRF token for the browser leg of the flow.

- The client persists `state` before redirecting to `/authorize`.
- The AS echoes it back on the callback.
- The client verifies it matches what it stored.

Skip `state` and you have a usable account-takeover bug: an attacker initiates an auth flow against their own account, sends the resulting callback URL to a victim, and the victim's client links the attacker's tokens to the victim's session.

PKCE and `state` together are the minimum.

## Practical guidance

- Use PKCE with `S256` *always*. `plain` exists in the spec but should not be used.
- Use `state`. Use [`nonce` for OIDC](../08-oidc.md).
- Pin the redirect URI to an exact string — no wildcards, no trailing-slash drift.
- For SPAs, never put refresh tokens in `localStorage`. Use HTTP-only cookies via a backend-for-frontend, or a service worker.
- Validate `iss` on the callback ([RFC 9207](../06-rfc-reference.md)) when your AS supports it — defends against mix-up attacks.

---

← [Flows overview](README.md) · ↑ [README](../../README.md) · → Next: [Implicit (deprecated)](implicit.md)
