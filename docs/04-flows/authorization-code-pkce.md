# 4.1 Authorization Code (+ PKCE)

> **In one line:** The standard, recommended way for a person to log in and grant an app access through their web browser.
>
> **Why it matters:** This is the one to use for almost anything involving a human. The rest of the page shows how it stays safe even if parts of it are intercepted.

**Who this is for:** every user-facing app today: web apps, SPAs, mobile, desktop. With PKCE, it's the only flow OAuth 2.1 endorses for human-driven access.

**Why it exists:** the authorization code is a single-use credential that's safe to pass through the user-agent (browser address bar), because exchanging it for an actual access token requires the client's secret (confidential clients) and/or the PKCE verifier (public clients).

---

## What PKCE is, in plain terms

PKCE (pronounced *"pixy"*) stands for **Proof Key for Code Exchange**, defined in **RFC 7636**. You can think of it as a small puzzle the client gives the Authorization Server to solve later, so that even if someone intercepts the authorization code in the middle of the flow, they can't actually redeem it.

The mental model: **the client invents a one-time password before starting the flow, sends a sealed envelope of it to the AS up front, and reveals the original password only when redeeming the code.** Anyone who steals the code from the browser still doesn't have the original password, so the stolen code is useless.

PKCE was originally designed in 2015 (RFC 7636) for mobile apps, but OAuth 2.1 makes it mandatory for **all** clients using authorization code, public and confidential alike. It's two extra parameters and one extra hash computation: near-zero cost for meaningful defence in depth.

## The attack PKCE prevents — a concrete walk-through

To see what PKCE actually defends against, let's follow one real attack scenario step by step. Then we'll re-run the exact same scenario with PKCE turned on and watch where it falls apart.

### The scenario

Philippe installs a banking app on his phone. He also has, unknowingly, a malicious "battery monitor" utility on the same phone that has registered itself as a handler for the URL scheme `bank://`. (Custom URL schemes on mobile are first-come-first-served on Android, and historically ambiguous on iOS: multiple apps can claim the same scheme.) The banking app uses `bank://callback` as its OAuth redirect URI.

```mermaid
sequenceDiagram
    autonumber
    participant U as Philippe
    participant BA as Banking App<br/>(legit)
    participant B as System Browser
    participant AS as Bank's Authorization Server
    participant MA as Malicious App<br/>(also installed)

    U->>BA: Tap "Sign in"
    BA->>B: Open bank://authorize?client_id=bank-app&...
    B->>AS: GET /authorize
    AS->>U: Show login page
    U->>AS: Enter credentials (legitimately)
    AS->>U: Show consent ("Bank App wants account.read")
    U->>AS: Approve
    AS->>B: 302 → bank://callback?code=XYZ
    Note over B,MA: OS resolves bank:// → multiple handlers exist.<br/>The malicious app responds first and wins.
    B->>MA: bank://callback?code=XYZ
    MA->>AS: POST /token<br/>grant_type=authorization_code<br/>&code=XYZ&client_id=bank-app
    AS->>MA: 200 { access_token, refresh_token }
    Note over MA: Attacker now holds the user's banking tokens.<br/>Banking app, meanwhile, sees nothing happen.
```

**What just happened, step by step:**

- Steps 1–7 are the *normal* authorization flow. Philippe really did sign in at his real bank. He really did approve the real banking app. Everything looks fine from his perspective.
- Step 8 is where it goes wrong. The AS sends a 302 redirect to `bank://callback?code=XYZ`. The OS sees a URL with the `bank://` scheme and has to pick an app to deliver it to. The malicious app is registered for that scheme too. Whichever app wins the OS lottery gets the redirect.
- Step 9: the OS hands the URL to the malicious app instead of the banking app.
- Steps 10–11, the malicious app does what *any* OAuth client does at this point: POST to `/token` with the code. The AS has no way to tell whether the caller is the real banking app or not, `client_id` is a public identifier, and there's no client secret in a mobile app (or if there is, it's extractable from the binary, so it's no secret either).
- The malicious app walks away with `access_token` + `refresh_token`. **The banking app never received the code at all**: Philippe might just see "sign-in failed, try again" with no idea anything was stolen.

This is the canonical attack PKCE was invented to stop. It was real enough that Apple and Google have since tightened URL-scheme handling, but it's still the cleanest illustration of why "the code in the redirect is enough" is a broken model for any client that can't keep a secret.

## How PKCE blocks the same attack

Now run the *exact same scenario* with PKCE turned on. The user, attacker, OS, and AS all behave identically. The only difference is that the banking app generates a per-flow secret before starting, sends only its hash to the AS, and holds the original in its own memory.

```mermaid
sequenceDiagram
    autonumber
    participant U as Philippe
    participant BA as Banking App<br/>(legit)
    participant B as System Browser
    participant AS as Bank's Authorization Server
    participant MA as Malicious App

    Note over BA: Generate code_verifier V<br/>(random 64-char string, kept in app memory)<br/>code_challenge = SHA256(V)
    U->>BA: Tap "Sign in"
    BA->>B: Open bank://authorize?<br/>code_challenge=SHA256(V)<br/>code_challenge_method=S256
    B->>AS: GET /authorize
    Note over AS: Issue code XYZ.<br/>Remember: code XYZ is bound to challenge SHA256(V).
    AS->>U: Login + consent
    U->>AS: Approve
    AS->>B: 302 → bank://callback?code=XYZ
    Note over B,MA: Same as before — malicious app<br/>wins the URL-scheme race.
    B->>MA: bank://callback?code=XYZ
    Note over MA: Attacker has the code, but NOT V.<br/>V was generated inside the banking app<br/>and never travelled anywhere over the wire.<br/>The challenge SHA256(V) is visible<br/>(it was in the /authorize URL), but you<br/>cannot reverse SHA-256 to recover V.
    MA->>AS: POST /token<br/>code=XYZ<br/>code_verifier=??? (attacker has to guess)
    AS->>AS: Compute SHA256(received verifier).<br/>Compare to stored challenge SHA256(V).<br/>They don't match.
    AS->>MA: 400 invalid_grant
    Note over MA: Attack fails. No token issued.
```

**What changed:**

- Steps 1–9 are identical. The attacker still wins the URL-scheme race and still has the code.
- Step 10 is where the world diverges. The malicious app POSTs to `/token`, same as before, but the AS now expects a `code_verifier` parameter. The attacker doesn't have V. The challenge value `SHA256(V)` was in the original `/authorize` URL (so the attacker could *see* it if they were watching), but a cryptographic hash is one-way: you cannot run it backwards to recover the input.
- Step 11: the AS hashes whatever the attacker sent, compares it to the challenge it stored at step 4, and finds they don't match.
- The AS rejects with `400 invalid_grant`. No token issued.

The legitimate banking app, by contrast, *does* have V (it generated it). When the banking app sees the redirect didn't come back to it (because the malicious app won), it simply never tries to redeem: there's no flow to complete. But if you imagine a scenario where the banking app *did* receive the code, it would send the verifier and the AS would accept. **The verifier is the proof that you're the same entity that started the flow.**

## Why the attacker can't just forge the verifier

A reader new to PKCE often asks: *"What stops the attacker from sending any random string as `code_verifier`?"*

Two things, working together:

1. **The challenge is committed to the AS up front.** When the legitimate client started the flow at `/authorize`, it sent `code_challenge = SHA256(V)`. The AS stored that challenge and bound it to the authorization code it later issues. So at `/token` time, the AS already knows the *exact* hash that needs to come out.
2. **SHA-256 is a one-way function.** Given the challenge `SHA256(V)`, there is no efficient way to find any input `V'` such that `SHA256(V') == SHA256(V)`. The only way to produce a verifier that matches is to know V, which only the legitimate client does, because V was generated locally and never transmitted.

So the attacker is stuck. They can guess random strings, but each guess has a vanishingly small probability of hashing to the right value. They can replay the challenge they saw in the URL, but the AS hashes whatever the client sends: sending the hash doesn't help, because `SHA256(SHA256(V)) ≠ SHA256(V)`.

### Does the length of the verifier matter? (the entropy angle)

Yes. The whole defence rests on the verifier being **unguessable**, which comes down to entropy. RFC 7636 requires `code_verifier` to be 43 to 128 characters from a fixed unreserved set, and the recommended recipe is 32 random bytes from a cryptographic random generator, Base64url-encoded into 43 characters carrying about 256 bits of entropy. To forge a match an attacker would have to either guess V outright (roughly 2^256 possibilities) or find some different `V'` that hashes to the same challenge (a SHA-256 preimage, also roughly 2^256). Both are far beyond what any amount of computing can reach.

Brute force is a non-issue for a second, independent reason: the attacker never gets a stream of guesses to grind against. The authorization code is **single-use and short-lived** (it expires in seconds to about a minute, and is invalidated the moment it is redeemed or a wrong verifier is presented), and the `/token` endpoint should be rate-limited. There is no oracle to hammer.

So the real risk is never the maths, it is a **badly generated verifier**: a weak or predictable random source, a too-short string, or using `code_challenge_method=plain` (where the challenge simply equals the verifier and protects nothing, which is why OAuth 2.1 forbids it). Generate V from a proper cryptographic random generator and use `S256`, and the attack surface is effectively zero.

## Other variants of the same family

The mobile URL-scheme hijack is the most vivid example, but the same defence works against several related attacks:

- **Permissive redirect-URI matching.** Some ASes historically allowed wildcard or prefix matches on `redirect_uri`. An attacker phishes the user into starting the flow with `https://evil.example.com/cb`, intercepts the code at their server, and redeems. PKCE breaks this the same way: no verifier means no token.
- **Referer / log / browser-history leakage.** The authorization code lives briefly in the URL bar. If the callback page makes a third-party request, the `Referer` header may carry the code. Web-server access logs capture full URLs. Browser history retains them. Anything that exfiltrates the URL gets the code, but PKCE makes that code unredeemable.
- **Authorization code injection.** A more subtle attack: attacker tricks a *legitimate client* into completing the flow with an authorization code that was actually issued for the attacker's session. Without PKCE, the legitimate client would happily redeem it and end up with the attacker's account. With PKCE, the legitimate client sends *its* verifier, the AS has stored the *attacker's* challenge, the hashes don't match, and the swap fails.

All of these reduce to the same principle: **PKCE binds the authorization code to the specific client that started the flow.** Anyone who didn't start the flow can't finish it.

## The crypto, in plain terms

PKCE uses three simple ingredients:

```mermaid
flowchart LR
    R["code_verifier<br/>random 43–128 chars<br/>kept secret on the client"] -->|SHA-256 + base64url| C["code_challenge<br/>sent in /authorize<br/>and stored by the AS"]
    R -.->|sent ONLY at /token,<br/>much later| V["AS verifies<br/>SHA-256(verifier) ==<br/>stored challenge"]
```

- **`code_verifier`**: a fresh random string the client generates for *each* authorization flow. 43 to 128 characters, URL-safe. Lives only in the client's memory.
- **`code_challenge`**: `BASE64URL(SHA-256(code_verifier))`. The client sends this to the AS at `/authorize`. The AS stores it alongside the issued authorization code.
- **`code_challenge_method`**: almost always `S256` (SHA-256). The spec also allows `plain` (where challenge == verifier), but `plain` is never the right answer; OAuth 2.1 forbids it.

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
- Pin the redirect URI to an exact string: no wildcards, no trailing-slash drift.
- For SPAs, never put refresh tokens in `localStorage`. Use HTTP-only cookies via a backend-for-frontend, or a service worker.
- Validate `iss` on the callback ([RFC 9207](../06-rfc-reference.md)) when your AS supports it: defends against mix-up attacks.

---

← [Flows overview](README.md) · ↑ [README](../../README.md) · → Next: [Implicit (deprecated)](implicit.md)
