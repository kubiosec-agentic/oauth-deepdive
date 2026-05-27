# OAuth 2.0 — A Deep-Dive Reference, with OAuth 2.1 and MCP

> Audience: technically literate readers comfortable with HTTP, RFCs, and protocol details. This guide is both a comprehensive reference and a practical primer — it tells you *what the protocol is*, *why it evolved the way it did*, *which flow to pick for which situation*, and *how OAuth 2.1 is being adapted for the Model Context Protocol (MCP)*.

---

## Table of contents

1. [What OAuth actually is (and isn't)](#1-what-oauth-actually-is-and-isnt)
2. [Core concepts and vocabulary](#2-core-concepts-and-vocabulary)
3. [The OAuth timeline — how we got here](#3-the-oauth-timeline--how-we-got-here)
4. [The flows, end to end (with HTTP)](#4-the-flows-end-to-end-with-http)
   - 4.1 [Authorization Code (+ PKCE)](#41-authorization-code--pkce)
   - 4.2 [Implicit (deprecated)](#42-implicit-deprecated)
   - 4.3 [Resource Owner Password Credentials (deprecated)](#43-resource-owner-password-credentials-deprecated)
   - 4.4 [Client Credentials](#44-client-credentials)
   - 4.5 [Refresh Token](#45-refresh-token)
   - 4.6 [Device Authorization Grant](#46-device-authorization-grant-rfc-8628)
   - 4.7 [JWT Bearer assertion grant](#47-jwt-bearer-assertion-grant-rfc-7523)
   - 4.8 [SAML 2.0 Bearer assertion grant](#48-saml-20-bearer-assertion-grant-rfc-7522)
   - 4.9 [Token Exchange](#49-token-exchange-rfc-8693)
   - 4.10 [CIBA](#410-ciba--client-initiated-backchannel-authentication)
5. [Tokens, in detail](#5-tokens-in-detail)
6. [The OAuth ecosystem — supporting RFCs you should know](#6-the-oauth-ecosystem--supporting-rfcs-you-should-know)
7. [OAuth 2.1 — what consolidated, what died](#7-oauth-21--what-consolidated-what-died)
8. [OAuth 2.1 for MCP — the dedicated section](#8-oauth-21-for-mcp--the-dedicated-section)
9. [Security considerations and common pitfalls](#9-security-considerations-and-common-pitfalls)
10. [Further reading](#10-further-reading)

---

## 1. What OAuth actually is (and isn't)

OAuth 2.0 is an **authorization-delegation framework**. Its job is to let a *client* (some application) obtain limited, scoped access to a *resource* (an API) on behalf of a *resource owner* (usually a human user), without that client ever seeing the resource owner's credentials.

What it explicitly is *not*:

- **It is not authentication.** OAuth tells an API "this request is allowed to read mailbox `philippe@…`." It does *not* tell the client *who the user is*. That's what **OpenID Connect (OIDC)** layers on top, by adding an `id_token` (a JWT about the user) and a `/userinfo` endpoint.
- **It is not a single protocol.** It's a framework. RFC 6749 leaves many choices (token format, opaque vs JWT, refresh-token rotation policy, signing algorithms…) to the implementer. That flexibility is both the source of OAuth's reach and the source of most of its security problems.
- **It is not "log in with Google."** That phrase is OIDC. People conflate them constantly because every consumer "social login" stack is OIDC on top of OAuth 2.0 authorization code.

The mental model that pays off: OAuth is about *handing your client a stamped ticket* that says "this much, for this long, against that API" — and never letting the API holding the ticket see your password.

---

## 2. Core concepts and vocabulary

| Term | Meaning |
|---|---|
| **Resource Owner** | The entity (usually a human) who owns the data and grants access. |
| **Client** | The application requesting access. Categorized as **public** (cannot keep a secret — SPAs, native apps) or **confidential** (can keep a secret — server-side web apps, machine services). |
| **Authorization Server (AS)** | Issues tokens after authenticating the resource owner and obtaining their consent. Operates `/authorize` and `/token` endpoints. |
| **Resource Server (RS)** | The API holding the protected data. Validates access tokens on every request. |
| **Access Token** | The credential the client uses on API calls. Short-lived (minutes to hours). Opaque or JWT. |
| **Refresh Token** | Long-lived credential used to obtain new access tokens without re-prompting the user. Confidential — never exposed to user agents. |
| **Authorization Grant** | The credential representing the resource owner's authorization. The "grant type" names the flow (code, client_credentials, etc.). |
| **Scope** | A space-separated list of strings (`read:mail write:calendar`) constraining what the token may do. |
| **Redirect URI** | The exact URL the AS sends the user back to after authorization. Must be pre-registered. |
| **Audience** | The intended recipient of the token (which RS may accept it). Set by `resource` parameter (RFC 8707) on the token request. |

A small but important nuance: the **client is the application**, not the user. "client_id" identifies the *application*, not whoever is using it. The resource owner is identified through the authorization step.

---

## 3. The OAuth timeline — how we got here

Understanding *why* OAuth 2.1 and the MCP profile look the way they do requires the lineage.

**2007 — OAuth 1.0.** Built at Twitter/Ma.gnolia, formalized as RFC 5849 in 2010. Used HMAC signatures on each request — every API call had to be cryptographically signed by the client. Secure but painful to implement; signature base-string construction was famously brittle.

**2012 — OAuth 2.0 (RFC 6749 + RFC 6750).** A complete rewrite by an IETF working group. The big bet: **TLS is now ubiquitous, so bearer tokens are fine** — drop request signing, send the token in `Authorization: Bearer …`, lean on HTTPS. Defined four canonical flows (Authorization Code, Implicit, Resource Owner Password Credentials, Client Credentials) plus the refresh-token flow. Eran Hammer, who chaired the working group, famously resigned and called the result "the road to hell" — too many extensibility points, too much left to implementers.

**2015 — PKCE (RFC 7636).** "Proof Key for Code Exchange." Patched a critical hole in the authorization-code flow for public clients (mobile, SPA): a malicious app on the same device could intercept the redirect URI and steal the authorization code. PKCE binds the code to a per-request secret the attacker can't replay. Initially mandatory only for native apps; the modern view is "always use it."

**2015 — Dynamic Client Registration (RFC 7591).** Lets a client POST its metadata to the AS and get a freshly minted `client_id` without a human creating an app registration in a dashboard. Critical for federations and (now) MCP.

**2018 — Authorization Server Metadata (RFC 8414).** The `/.well-known/oauth-authorization-server` document. Discovery — clients can find token endpoints, supported scopes, signing keys without hard-coding them. Mirrors the OIDC `/.well-known/openid-configuration` doc that has existed since 2014.

**2020 — Resource Indicators for OAuth 2.0 (RFC 8707).** The `resource` parameter on the authorization and token requests. Lets the client say "this token is for `https://api.example.com`" — the AS can then bind the token's audience, and the RS rejects tokens minted for somebody else. Critical for the multi-tenant world MCP lives in.

**2020 — OAuth 2.0 Security Best Current Practice** (published as RFC 9700 in January 2025). Tightened the screws: PKCE for all clients, exact redirect-URI matching, refresh-token rotation, drop Implicit, drop Password grant.

**2020 → ongoing — OAuth 2.1 (draft-ietf-oauth-v2-1).** Consolidation. Folds 6749, 6750, PKCE, BCP recommendations into one readable document. As of **draft-15 (March 2026)** it's still an Internet Draft on standards track, expected to obsolete RFC 6749 and RFC 6750.

**2023 — DPoP (RFC 9449).** "Demonstrating Proof-of-Possession." Sender-constrained tokens at the application layer. The client signs a small JWT (`DPoP` header) per request proving it holds the key the token was issued to. Tokens stolen from logs or memory no longer work in another client. The lightweight alternative to mutual TLS.

**2025 — Protected Resource Metadata (RFC 9728).** The mirror of RFC 8414 but for the resource server: `/.well-known/oauth-protected-resource` advertises *which authorization servers* are trusted for *which audiences*. This is the keystone that lets MCP servers decouple from any particular AS.

**2024 → 2026 — MCP authorization specification.** First MCP spec (2024-11-05) had no formal auth profile. The 2025-03-26 revision picked OAuth 2.1. The 2025-06-18 revision split the MCP server role from the AS role (MCP server is purely a resource server, points clients at an external AS via RFC 9728). The 2025-11-25 spec further tightened resource-parameter handling. The 2026-07-28 release candidate is the largest revision since launch and aligns more closely with mainstream OAuth/OIDC deployments.

The throughline: **every revision narrows the framework's optionality and pushes toward sender-constrained, audience-bound, PKCE-protected, discovery-driven OAuth.** That's the world MCP was designed into.

---

## 4. The flows, end to end (with HTTP)

A note on notation: lines beginning with `>` are sent by the client, lines beginning with `<` are responses from the server. Real-world examples use HTTPS exclusively — TLS is non-negotiable in OAuth 2.0+.

### 4.1 Authorization Code (+ PKCE)

**Who this is for:** every user-facing app today — web apps, SPAs, mobile, desktop. With PKCE, it's the only flow OAuth 2.1 endorses for human-driven access.

**Why it exists:** the authorization code is a single-use credential that's safe to pass through the user-agent (browser address bar), because exchanging it for an actual access token requires the client's secret (confidential clients) and/or the PKCE verifier (public clients).

**The dance:**

1. Client generates a `code_verifier` (random 43–128 chars) and derives `code_challenge = BASE64URL(SHA256(code_verifier))`.
2. Client redirects the user to `/authorize` with `response_type=code`, `code_challenge`, `code_challenge_method=S256`.
3. AS authenticates the user, gets consent, redirects back with `?code=…&state=…`.
4. Client POSTs to `/token` with the `code` *and* the original `code_verifier`.
5. AS verifies `SHA256(code_verifier) == code_challenge`, returns access (and optionally refresh) token.

**HTTP — step 2, the authorization request:**

```
> GET /authorize?
    response_type=code
    &client_id=s6BhdRkqt3
    &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
    &scope=read%3Amail%20write%3Acalendar
    &state=xyzABC123
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
    &resource=https%3A%2F%2Fapi.example.com
  Host: as.example.com
```

After the user authenticates and consents:

```
< HTTP/1.1 302 Found
  Location: https://client.example.com/cb?
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=xyzABC123
```

**Step 4 — the token exchange:**

```
> POST /token HTTP/1.1
  Host: as.example.com
  Content-Type: application/x-www-form-urlencoded
  Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW   # client_id:client_secret, confidential only

  grant_type=authorization_code
  &code=SplxlOBeZQQYbYS6WxSbIA
  &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
  &resource=https%3A%2F%2Fapi.example.com
```

```
< HTTP/1.1 200 OK
  Content-Type: application/json
  Cache-Control: no-store

  {
    "access_token": "eyJhbGciOiJSUzI1NiIs...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
    "scope": "read:mail write:calendar"
  }
```

The `state` parameter is your CSRF token for the browser leg — the client must persist it before redirect and verify it on callback. Skip this and you have a usable account-takeover bug.

**Practical guidance:** Use PKCE with `S256` *always*, even for confidential clients. Use `state`. Pin the redirect URI to an exact string (no wildcards, no trailing-slash drift). For SPAs, never put refresh tokens in `localStorage` — store them in HTTP-only cookies via a tiny backend, or use a service worker.

---

### 4.2 Implicit (deprecated)

**Who this was for:** browser-side SPAs before CORS was widespread on `/token` endpoints. The AS returned the access token directly in the redirect fragment (`#access_token=…`), skipping the back-channel exchange.

**Why it's dead:**

- Access tokens land in browser history, server logs, Referer headers.
- No code-to-token exchange means no client authentication and (worse) no PKCE protection.
- Refresh tokens were never issued, so SPAs hit silent-renewal hacks via hidden iframes.

OAuth 2.1 omits it. The migration path is Authorization Code + PKCE (now supported with CORS at every major AS).

For reference, the request looked like:

```
> GET /authorize?response_type=token&client_id=…&redirect_uri=…&state=…
```

If you find this in a codebase today, treat it as a P1 security bug.

---

### 4.3 Resource Owner Password Credentials (deprecated)

**Who this was for:** "trusted first-party" apps where the user could type their password into the client itself, which then POSTed it to the AS:

```
> POST /token
  grant_type=password
  &username=philippe
  &password=hunter2
  &client_id=…
```

**Why it's dead:** the entire point of OAuth was that the client never sees credentials. This flow violated that. Worse, it normalized phishing — users learned to type their primary password into anything labelled "login." MFA, federation, and policy controls at the AS are all bypassed.

OAuth 2.1 omits it. All major IdPs have either removed it or restricted it to legacy migration paths.

---

### 4.4 Client Credentials

**Who this is for:** machine-to-machine. No human, no user context. A service authenticates *as itself*.

```
> POST /token HTTP/1.1
  Host: as.example.com
  Content-Type: application/x-www-form-urlencoded
  Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

  grant_type=client_credentials
  &scope=metrics:read
  &resource=https%3A%2F%2Fapi.example.com
```

```
< HTTP/1.1 200 OK
  {
    "access_token": "2YotnFZFEjr1zCsicMWpAA",
    "token_type": "Bearer",
    "expires_in": 3600
  }
```

There is no refresh token here — the credentials *are* the refresh mechanism. The client simply re-hits `/token` when its token nears expiry.

**Practical guidance:** never use Client Credentials to "stand in for" a user. The token has no `sub` claim, no user context — using it to read user data is a guaranteed authorization-bypass bug. For machine identities, prefer **Workload Identity Federation** + `urn:ietf:params:oauth:grant-type:jwt-bearer` (see §4.7) over long-lived `client_secret`s.

---

### 4.5 Refresh Token

Not a standalone flow — a continuation. The client trades a refresh token for a fresh access token (and ideally a rotated refresh token):

```
> POST /token
  Authorization: Basic …

  grant_type=refresh_token
  &refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
  &scope=read:mail            # may downscope, may not upscope
```

```
< 200 OK
  {
    "access_token": "…",
    "expires_in": 3600,
    "refresh_token": "NEW-rotated-value"     # rotation
  }
```

**Refresh-token rotation** is the modern default: every use returns a new refresh token and invalidates the old one. If an old refresh token is ever used again, the AS treats it as theft, revokes the family, and forces re-authentication. This is the only protection that meaningfully limits the blast radius of a stolen refresh token for a public client.

---

### 4.6 Device Authorization Grant (RFC 8628)

**Who this is for:** input-constrained devices — TVs, CLIs, IoT — that can display a code but can't reasonably host a browser-redirect flow.

1. Device POSTs to `/device_authorization`:

```
> POST /device_authorization
  client_id=…&scope=…
```

```
< 200 OK
  {
    "device_code":      "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
    "user_code":        "WDJB-MJHT",
    "verification_uri": "https://example.com/device",
    "verification_uri_complete":
                        "https://example.com/device?user_code=WDJB-MJHT",
    "expires_in":       1800,
    "interval":         5
  }
```

2. Device shows the user: *"Visit https://example.com/device and enter WDJB-MJHT"*.
3. Meanwhile, the device polls `/token`:

```
> POST /token
  grant_type=urn:ietf:params:oauth:grant-type:device_code
  &device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
  &client_id=…
```

Returns `authorization_pending` until the user finishes, then a normal token response.

**Used by:** the `gh` CLI, the `aws sso login` flow, Apple TV apps, smart-TV streaming, modern terminal-based OAuth.

---

### 4.7 JWT Bearer assertion grant (RFC 7523)

The client presents a *signed JWT* as its grant. Used heavily in cloud federation:

```
> POST /token
  grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
  &assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjE2In0.…
```

The JWT proves the caller's identity via a key the AS already trusts (often via an OIDC-discoverable JWKS — this is how AWS IAM Roles for Service Accounts, GitHub Actions OIDC, and most CI→Cloud federations work). Replaces long-lived shared secrets with short-lived, audience-bound assertions.

---

### 4.8 SAML 2.0 Bearer assertion grant (RFC 7522)

The same pattern as 4.7, but the assertion is a SAML 2.0 token instead of a JWT. Mostly seen in enterprise bridges between legacy SAML SSO and modern OAuth-based APIs. New systems should prefer JWT bearer.

---

### 4.9 Token Exchange (RFC 8693)

Trade one token for another. The grant type is `urn:ietf:params:oauth:grant-type:token-exchange`. Two patterns dominate:

- **Impersonation:** a service holds the user's token and exchanges it for a narrower-scoped token to call a downstream service. The downstream sees the original user as `sub`.
- **Delegation:** the new token carries an `act` (actor) claim identifying the service acting on behalf of the user — auditable, distinguishable from direct user access.

```
> POST /token
  grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  &subject_token=eyJ…
  &subject_token_type=urn:ietf:params:oauth:token-type:access_token
  &audience=https://downstream.example.com
  &requested_token_type=urn:ietf:params:oauth:token-type:access_token
  &scope=read:profile
```

This is the bedrock of microservice OAuth and is increasingly relevant to **AI-agent scenarios** — an agent that needs to call N tools on a user's behalf should mint per-tool narrowly-scoped tokens via Token Exchange, not pass the same broad-scope token to every tool.

---

### 4.10 CIBA — Client-Initiated Backchannel Authentication

OpenID Foundation spec, not strictly OAuth, but increasingly relevant. The client initiates auth *without* the user being at the same device — the AS pushes a notification to the user's phone, the user approves, and the client polls (or receives a ping) at `/token`. Used in banking, call-center scenarios, and "approve on your phone" UX. Worth knowing exists; not yet common in MCP.

---

## 5. Tokens, in detail

### Bearer tokens (RFC 6750)

The default. Send `Authorization: Bearer <token>`. Whoever has the token can use it. Security model leans entirely on TLS and the assumption that the token never leaks. This was OAuth 2.0's original compromise — drop request signing for protocol simplicity — and most token-theft incidents trace back to it.

### Sender-constrained tokens

Two production-grade options bind a token to the key that requested it:

- **mutual TLS (RFC 8705).** The TLS client certificate's thumbprint is embedded in the token (`cnf.x5t#S256`). RS verifies that the connection's client cert matches. Strongest, but requires mTLS infrastructure end-to-end.
- **DPoP (RFC 9449).** The client signs a `DPoP` header per request with a JWK it bound to the token at issuance. RS verifies the JWT, the `htm`/`htu` claims (HTTP method and URL), and the `jti` (replay protection). Works through plain HTTPS and through proxies; much easier to deploy than mTLS.

Sender-constraint is rapidly becoming the expectation for high-value APIs. Open-banking, FAPI 2.0, and emerging AI-agent profiles all require it.

### JWT access tokens (RFC 9068)

Stating the obvious: an opaque access token is just a string; a JWT access token is a signed JSON document. RFC 9068 standardizes the claims:

```
{
  "iss": "https://as.example.com",
  "sub": "user-7b8c…",
  "aud": "https://api.example.com",
  "client_id": "s6BhdRkqt3",
  "iat": 1748352000,
  "nbf": 1748352000,
  "exp": 1748355600,
  "jti": "9d2…",
  "scope": "read:mail",
  "cnf": { "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I" }
}
```

JWT access tokens let the RS validate the token without a round-trip to the AS — at the cost of any revocation being delayed to token expiry. Pick short lifetimes (5–15 min) and lean on the `jti` + a deny-list if you need immediate revocation.

### Token Introspection (RFC 7662)

The opposite trade-off — opaque token, RS asks the AS:

```
> POST /introspect
  Authorization: Basic …

  token=mF_9.B5f-4.1JqM
```

```
< 200 OK
  { "active": true, "scope": "read:mail", "sub": "user-7b8c", "exp": 1748355600 }
```

Use when you need real-time revocation and centralized policy. Cost: latency, and the AS becomes a hot path.

---

## 6. The OAuth ecosystem — supporting RFCs you should know

| RFC | What it gives you |
|---|---|
| **6749** | OAuth 2.0 framework. The "main" spec. |
| **6750** | Bearer token usage. |
| **7009** | Token revocation (`/revoke`). |
| **7517** | JWK / JWKS. |
| **7519** | JWT. |
| **7521** | Generic assertion framework. |
| **7522** | SAML 2.0 assertion grant. |
| **7523** | JWT assertion grant. |
| **7591** | Dynamic Client Registration. |
| **7592** | Dynamic Client Management. |
| **7636** | PKCE. |
| **7662** | Token Introspection. |
| **8252** | OAuth 2.0 for Native Apps (BCP 212). |
| **8414** | Authorization Server Metadata (`/.well-known/oauth-authorization-server`). |
| **8628** | Device Authorization Grant. |
| **8693** | Token Exchange. |
| **8705** | Mutual TLS client authentication and sender-constrained tokens. |
| **8707** | Resource Indicators (`resource` parameter). |
| **9068** | JWT Profile for OAuth 2.0 Access Tokens. |
| **9101** | JWT-Secured Authorization Request (JAR). |
| **9126** | Pushed Authorization Requests (PAR). |
| **9207** | `iss` parameter on the authorization response. |
| **9449** | DPoP. |
| **9470** | Step-Up Authentication Challenge. |
| **9700** | OAuth 2.0 Security Best Current Practice (BCP). |
| **9728** | OAuth 2.0 Protected Resource Metadata (`/.well-known/oauth-protected-resource`). |

If you have one slot in your reading list after RFC 6749, make it **RFC 9700** — it's the security state of the art compressed into one document, and it's what OAuth 2.1 is built from.

---

## 7. OAuth 2.1 — what consolidated, what died

OAuth 2.1 (draft-ietf-oauth-v2-1, currently at **-15** as of March 2026, intended status: Standards Track) does not invent new mechanisms. It is a **rewrite that obsoletes RFC 6749 and RFC 6750** by folding in the best-practice consensus that had accumulated:

**Now mandatory:**

- **PKCE for all clients** using the authorization code flow — confidential and public alike. (RFC 6749 made it optional; RFC 9700 made it BCP; 2.1 makes it normative.)
- **Exact-string redirect-URI matching.** No wildcards, no prefix matching, no `localhost`-special-casing.
- **HTTPS** for all endpoints. Always was best practice; now spec.
- **No password grant.**
- **No implicit grant.**

**Still permitted but tightened:**

- Bearer tokens, but with explicit guidance toward sender-constraint where the threat model warrants.
- Refresh tokens, with rotation strongly recommended for public clients.

**Carried in from supporting RFCs:**

- `state` (CSRF) on authorization requests.
- `iss` (RFC 9207) on authorization responses, to defend against mix-up attacks.
- Resource indicators (RFC 8707) when the AS serves multiple resources.

What this means practically: **if you're implementing OAuth today, target OAuth 2.1 and you will be on the right side of every security review for the next decade.** The draft is not yet a Standards-Track RFC, but its substance is already incorporated in major frameworks (Spring Authorization Server, Keycloak, Auth0, Entra, etc.), and the FAPI 2.0 and MCP profiles both reference it normatively.

---

## 8. OAuth 2.1 for MCP — the dedicated section

### 8.1 What MCP is, in one paragraph

The **Model Context Protocol (MCP)** is an open protocol, originally from Anthropic, that defines how AI applications (clients — e.g., Claude Desktop, an IDE plugin, an agent) connect to external tools and data sources (servers — e.g., a GitHub MCP server, a database MCP server). MCP servers expose **tools**, **resources**, and **prompts**; clients negotiate capabilities and invoke them over JSON-RPC carried by stdio, SSE, or **Streamable HTTP**. As soon as MCP servers started exposing real user data over a network transport, the question "how do we authorize this?" became unavoidable — and OAuth 2.1 is the answer the MCP working group converged on.

### 8.2 The role split — MCP server as resource server

The single most important architectural decision in the MCP authorization spec is that **the MCP server is purely an OAuth 2.1 Resource Server**. It does *not* mint tokens, does *not* run an authorization endpoint, does *not* know how to authenticate users. It delegates all of that to a separate Authorization Server — which can be Entra ID, Okta, Auth0, Keycloak, WorkOS, a homegrown AS, or anything else that speaks the standards.

This is a deliberate evolution. The original 2025-03-26 spec allowed the MCP server to also be its own AS. The 2025-06-18 revision **split the roles** because conflating them led to under-secured MCP servers reinventing identity. Today:

- **MCP client** ≈ OAuth 2.1 client (typically *public*, since the AI host application can't reliably keep secrets).
- **MCP server** ≈ OAuth 2.1 resource server.
- **Authorization server** ≈ any conformant AS, advertised by the MCP server via RFC 9728.

### 8.3 The full handshake

This is the canonical sequence, end to end:

```
┌──────────┐        ┌──────────────┐        ┌──────────────────┐
│ MCP      │        │ MCP Server   │        │ Authorization    │
│ Client   │        │ (Resource)   │        │ Server           │
└────┬─────┘        └──────┬───────┘        └────────┬─────────┘
     │                     │                         │
     │ 1) MCP request without token                   │
     ├────────────────────►│                         │
     │                     │                         │
     │ 2) 401 + WWW-Authenticate:                    │
     │    Bearer resource_metadata="…/.well-known/   │
     │    oauth-protected-resource"                  │
     │◄────────────────────┤                         │
     │                     │                         │
     │ 3) GET .well-known/oauth-protected-resource   │
     ├────────────────────►│                         │
     │ 4) { authorization_servers: [https://as…] }   │
     │◄────────────────────┤                         │
     │                     │                         │
     │ 5) GET as/.well-known/oauth-authorization-server         │
     ├──────────────────────────────────────────────►│
     │ 6) AS metadata (RFC 8414)                                │
     │◄──────────────────────────────────────────────┤
     │                     │                         │
     │ 7) POST /register (DCR — RFC 7591) if needed             │
     ├──────────────────────────────────────────────►│
     │ 8) { client_id }                                         │
     │◄──────────────────────────────────────────────┤
     │                     │                         │
     │ 9) Authorization Code + PKCE flow (browser leg)          │
     │     resource=https://mcp.example.com (RFC 8707)          │
     ├──────────────────────────────────────────────►│
     │ 10) access_token (audience-bound to MCP server)          │
     │◄──────────────────────────────────────────────┤
     │                     │                         │
     │ 11) MCP request, Authorization: Bearer …      │
     ├────────────────────►│                         │
     │                     │ 12) verify token        │
     │                     │     (sig + aud + iss)   │
     │ 13) MCP response    │                         │
     │◄────────────────────┤                         │
```

### 8.4 The discovery chain (RFC 9728 → RFC 8414)

When a client hits an MCP server without a (valid) token, the server returns:

```
< HTTP/1.1 401 Unauthorized
  WWW-Authenticate: Bearer
    resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"
```

The client fetches that document — defined by **RFC 9728**:

```
< GET /.well-known/oauth-protected-resource HTTP/1.1
  Host: mcp.example.com

> 200 OK
  {
    "resource": "https://mcp.example.com",
    "authorization_servers": [
      "https://login.example.com"
    ],
    "scopes_supported": ["mcp:tools.read", "mcp:tools.invoke"],
    "bearer_methods_supported": ["header"],
    "resource_documentation": "https://mcp.example.com/docs"
  }
```

The client then resolves the AS via **RFC 8414**:

```
< GET /.well-known/oauth-authorization-server HTTP/1.1
  Host: login.example.com

> 200 OK
  {
    "issuer": "https://login.example.com",
    "authorization_endpoint": "https://login.example.com/authorize",
    "token_endpoint":         "https://login.example.com/token",
    "registration_endpoint":  "https://login.example.com/register",
    "code_challenge_methods_supported": ["S256"],
    "grant_types_supported": ["authorization_code", "refresh_token"],
    "response_types_supported": ["code"],
    "token_endpoint_auth_methods_supported": ["none", "client_secret_basic"]
  }
```

The chain is the load-bearing part of the design: **the MCP server tells the client which AS to trust, and the AS describes itself.** Nothing is hard-coded in the client.

### 8.5 Dynamic Client Registration in MCP

The catch: MCP clients are unknown to the AS until they try to connect. There is no human in the loop to provision an OAuth app for "Claude Desktop talking to your GitHub MCP server" before the conversation starts.

**RFC 7591 — Dynamic Client Registration** is the answer:

```
> POST /register HTTP/1.1
  Host: login.example.com
  Content-Type: application/json

  {
    "client_name": "Claude Desktop (Philippe's laptop)",
    "redirect_uris": ["http://127.0.0.1:51247/cb"],
    "grant_types": ["authorization_code", "refresh_token"],
    "response_types": ["code"],
    "token_endpoint_auth_method": "none",
    "scope": "mcp:tools.read mcp:tools.invoke"
  }
```

```
< 201 Created
  {
    "client_id": "mcp-cli-abc123",
    "client_id_issued_at": 1748352000,
    "redirect_uris": ["http://127.0.0.1:51247/cb"],
    "grant_types": ["authorization_code", "refresh_token"]
  }
```

The MCP spec says clients **MAY** support DCR (i.e., it's optional but expected for most deployments). Enterprise IdPs that don't allow open DCR typically gate it behind a software statement or an admin-provisioned **initial access token** — RFC 7591 supports both.

### 8.6 Resource indicators — non-negotiable in MCP

**This is the rule that surprises people:** MCP clients **MUST** include the `resource` parameter (RFC 8707) on both the authorization request and the token request, set to the canonical URI of the MCP server they intend to call.

```
> POST /token
  …
  &resource=https%3A%2F%2Fmcp.example.com
```

The MCP server **MUST** validate that the access token's audience matches *its own* canonical URI. Without this, a token issued for `mcp-server-A` would also work at `mcp-server-B` if both happened to trust the same AS — a **confused-deputy** attack, and a real one in the MCP ecosystem because users frequently connect to many MCP servers through one AS.

A token presented at the MCP server should look (decoded) like:

```
{
  "iss": "https://login.example.com",
  "sub": "user-7b8c…",
  "aud": "https://mcp.example.com",   // ← bound by RFC 8707
  "client_id": "mcp-cli-abc123",
  "scope": "mcp:tools.invoke",
  "exp": 1748355600
}
```

If `aud` doesn't match the MCP server's canonical URI, the server **MUST** reject with 401. This single check eliminates a whole class of cross-MCP token-replay attacks.

### 8.7 What an MCP server actually has to implement

Concretely, the minimum bar for a compliant MCP authorization deployment:

- Serve `/.well-known/oauth-protected-resource` (RFC 9728).
- On every request, validate the bearer token:
  1. Signature against the AS's JWKS (or call `/introspect`).
  2. `iss` == one of the trusted authorization servers from your PRM doc.
  3. `aud` == this MCP server's canonical URI.
  4. `exp` and `nbf`.
  5. Required scopes are present.
- Return `401` with `WWW-Authenticate: Bearer resource_metadata="…"` for unauthenticated requests.
- Return `403` with `error="insufficient_scope"` for missing scopes.

The AS does the heavy lifting (login, MFA, consent, token issuance, DCR). The MCP server's auth code is mostly *token validation* — a few hundred lines, not a few thousand.

### 8.8 Why this profile is interesting beyond MCP

The MCP authorization spec is the first widely-adopted use case of a **fully discovery-driven, audience-bound, PKCE-only, DCR-friendly OAuth 2.1 deployment** in the wild. It's the practical proof that OAuth 2.1 + RFC 8707 + RFC 9728 + RFC 7591 compose cleanly. Expect to see the same pattern replicated in the next wave of agent-to-tool standards.

Two open research-leaning questions worth tracking:

- **Token Exchange (RFC 8693) for agent fan-out.** When an agent invokes ten tools on a user's behalf, should it present the same broad token to each, or mint ten narrowly-scoped child tokens via Token Exchange? The MCP spec doesn't mandate this yet but the pattern is appearing in enterprise deployments.
- **DPoP for MCP.** Bearer-only is the default today, but sender-constrained tokens via DPoP (RFC 9449) would meaningfully harden long-running agent sessions where access tokens linger in process memory. The 2026-07-28 release candidate touches this territory.

### 8.9 Common MCP-auth pitfalls

- **Hard-coding the AS in the client.** Defeats the whole discovery chain. Always read `authorization_servers` from PRM.
- **Skipping the `resource` parameter.** Your token becomes usable at any MCP server the AS issues for. Confused-deputy bait.
- **Trusting the token's `iss` blindly.** The MCP server must check `iss` against *its own* PRM-declared trusted list, not against whatever the token says.
- **Storing refresh tokens in client storage that's accessible to plugin sandboxes.** Many MCP client hosts run untrusted code. Treat the refresh token like a long-lived password.
- **Using `client_secret` in an MCP CLI.** A CLI is a public client. Use PKCE with `token_endpoint_auth_method: none`.
- **No revocation path.** Users must be able to revoke an MCP client's access from the AS console — this is *not* the MCP server's job, but the MCP server should respect it (don't cache token validity beyond `exp`).

---

## 9. Security considerations and common pitfalls

Even with OAuth 2.1, you can build something insecure. The recurring failure modes:

**Redirect-URI handling.** Wildcard matching, allowing user-controlled hosts, or failing to compare exact-string. Once an attacker controls the redirect, they own the auth code (and PKCE only helps if they can't also coerce the legitimate verifier).

**Mix-up attacks.** A malicious AS in a multi-AS deployment can trick a client into completing the flow with the wrong AS. **Defense:** RFC 9207 — the AS includes `iss` in the authorization response, and the client verifies it before exchanging the code.

**Token leakage via Referer / logs / browser history.** Why Implicit died. Why query-string tokens are forbidden in OAuth 2.1.

**Public-client refresh-token theft.** Without rotation, a stolen refresh token works until it expires. With rotation, the legitimate client's next refresh trips the alarm.

**Scope creep.** "Read all data" scopes get rubber-stamped because consent UIs are bad. Design scopes around *actions* (`mail:read:unread`, not `full_mailbox`).

**Confused deputy across resource servers.** Solved by RFC 8707 + audience validation. The MCP profile is explicit about this; many older deployments are not.

**Open redirector chains.** Combine a misconfigured redirect URI with an open redirector on the same host and you have a token-exfiltration primitive. Don't let your hostname host both an OAuth client and unrelated user-content URLs.

**MFA bypass via legacy flows.** If you've turned off Password grant in your AS, audit your tenants — large orgs sometimes keep it enabled for "legacy migration" indefinitely.

**Sender-constraint coverage gaps.** DPoP only protects requests that present the DPoP header; an old code path that skips it negates the protection. All-or-nothing.

---

## 10. Further reading

Specs (chronological by relevance):

- **RFC 6749** — OAuth 2.0 Authorization Framework
- **RFC 6750** — Bearer Token Usage
- **RFC 7636** — PKCE
- **RFC 7591** — Dynamic Client Registration
- **RFC 8414** — Authorization Server Metadata
- **RFC 8707** — Resource Indicators
- **RFC 9068** — JWT Profile for Access Tokens
- **RFC 9449** — DPoP
- **RFC 9700** — OAuth 2.0 Security Best Current Practice (BCP)
- **RFC 9728** — Protected Resource Metadata
- **draft-ietf-oauth-v2-1** — OAuth 2.1 (current draft -15, March 2026)

MCP authorization:

- [MCP specification — Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization) — the canonical spec
- [The 2026-07-28 MCP Specification Release Candidate](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/) — what's coming
- [MCP, OAuth 2.1, PKCE, and the Future of AI Authorization](https://aembit.io/blog/mcp-oauth-2-1-pkce-and-the-future-of-ai-authorization/) — accessible overview
- [Technical Deconstruction of MCP Authorization](https://kane.mx/posts/2025/mcp-authorization-oauth-rfc-deep-dive/) — RFC-by-RFC walkthrough

Books and long-form:

- *OAuth 2 in Action* — Justin Richer and Antonio Sanso. Still the best book on the framework.
- [oauth.net](https://oauth.net/2.1/) — Aaron Parecki's curated reference, kept current.

---

*If you'd like a follow-up: a working MCP authorization implementation walkthrough (server-side code in your language of choice, full discovery + token-validation flow), a security review of an existing OAuth deployment, or a deeper dive on agent-specific patterns like Token Exchange fan-out — just ask.*
