# 4.4 Client Credentials

> **In one line:** How one program talks to another with no person involved: a service proving who it is.
>
> **Why it matters:** Common for background jobs and automation. The big trap, covered here, is accidentally using it to act for a user it cannot really represent.

**Who this is for:** machine-to-machine. No human, no user context. A service authenticates *as itself*.

## The sequence

```mermaid
sequenceDiagram
    autonumber
    participant C as Service / Client
    participant AS as Authorization Server
    participant RS as Resource Server

    C->>AS: POST /token<br/>grant_type=client_credentials<br/>Basic auth + scope + resource
    AS->>C: 200 { access_token, expires_in }
    C->>RS: GET /resource<br/>Authorization: Bearer ...
    RS->>C: 200 data
    Note over C,AS: When token expires,<br/>just hit /token again
```

## HTTP

```http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=client_credentials
&scope=metrics:read
&resource=https%3A%2F%2Fapi.example.com
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type":   "Bearer",
  "expires_in":   3600
}
```

There is no refresh token here: the credentials *are* the refresh mechanism. The client simply re-hits `/token` when its token nears expiry.

## Practical guidance

**Never use Client Credentials to "stand in for" a user.** The token has no `sub` claim (or has `sub == client_id`), no user context: using it to read user data is a guaranteed authorization-bypass bug. If your AI agent uses a Client Credentials token to read mailbox data on behalf of a user, your audit log will not be able to distinguish the user from the agent and any human attacker who has compromised the service.

**For machine identities, prefer workload identity federation** + [`urn:ietf:params:oauth:grant-type:jwt-bearer`](jwt-bearer.md) over long-lived `client_secret` values. Examples: GitHub Actions OIDC → cloud, Kubernetes ServiceAccount → cloud, SPIFFE/SPIRE → API.

**Don't sprinkle `client_secret` into config files**, environment variables in shared CI, or container images. If you must use Client Credentials, rotate the secret automatically and short-lived.

---

← [Password (deprecated)](password.md) · ↑ [Flows](README.md) · → Next: [Refresh Token](refresh-token.md)
