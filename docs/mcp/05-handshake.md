# 9.5 The full handshake, end to end

This is everything from the preceding pages, stitched together into one sequence.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as MCP Client<br/>(public OAuth 2.1 client)
    participant RS as MCP Server<br/>(Resource Server)
    participant AS as Authorization Server

    Note over C,RS: --- Discovery ---
    C->>RS: MCP request (no token)
    RS->>C: 401 + WWW-Authenticate:<br/>Bearer resource_metadata="..."
    C->>RS: GET /.well-known/oauth-protected-resource
    RS->>C: { authorization_servers: [AS], scopes_supported }
    C->>AS: GET /.well-known/oauth-authorization-server
    AS->>C: { authorization_endpoint, token_endpoint,<br/>registration_endpoint, jwks_uri }

    Note over C,AS: --- Registration (DCR, optional) ---
    C->>AS: POST /register<br/>{ client_name, redirect_uris, ... }
    AS->>C: 201 { client_id }

    Note over U,AS: --- Authorization (browser leg) ---
    C->>C: Generate code_verifier + state + nonce
    C->>U: Open browser →<br/>/authorize?response_type=code<br/>&code_challenge=...<br/>&state=...<br/>&resource=https://mcp.example.com<br/>&scope=mcp:tools.invoke
    U->>AS: Login, MFA, consent
    AS->>U: 302 → redirect_uri?code=...&state=...

    Note over C,AS: --- Token exchange ---
    C->>AS: POST /token<br/>grant_type=authorization_code<br/>&code=...&code_verifier=...<br/>&resource=https://mcp.example.com
    AS->>C: { access_token aud=mcp.example.com,<br/>refresh_token, expires_in }

    Note over C,RS: --- MCP calls ---
    C->>RS: MCP request<br/>Authorization: Bearer ...
    RS->>RS: Validate: sig, iss in PRM list,<br/>aud == self, exp, scope
    RS->>C: MCP response

    Note over C,AS: --- When token expires ---
    C->>AS: POST /token grant_type=refresh_token
    AS->>C: { rotated access_token + refresh_token }
```

## What's happening in plain English

1. **Discovery.** The client knows only the MCP server URL. It pulls PRM, pulls AS metadata, and now knows every endpoint it needs.
2. **Registration.** If the client doesn't already have a `client_id` for this AS, it registers itself via DCR.
3. **Authorization.** Standard OAuth 2.1 authorization code + PKCE in the browser. The user sees a consent screen for the MCP server's scopes. The `resource` parameter pins the token's audience.
4. **Token exchange.** The code is exchanged for an audience-bound access token.
5. **MCP calls.** The client uses the token on the MCP server. The server validates everything on every call.
6. **Refresh.** When the access token expires, the refresh token gets a new one (with rotation).

## The HTTP, all in one place

### Trigger: client tries MCP without a token

```http
POST /mcp HTTP/1.1
Host: mcp.example.com
Content-Type: application/json

{"jsonrpc":"2.0","method":"tools/list","id":1}
```

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
    resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"
```

### Discovery

```http
GET /.well-known/oauth-protected-resource HTTP/1.1
Host: mcp.example.com
```

```http
HTTP/1.1 200 OK
{
  "resource":              "https://mcp.example.com",
  "authorization_servers": ["https://login.example.com"],
  "scopes_supported":      ["mcp:tools.read", "mcp:tools.invoke"]
}
```

```http
GET /.well-known/oauth-authorization-server HTTP/1.1
Host: login.example.com
```

```http
HTTP/1.1 200 OK
{
  "issuer":                  "https://login.example.com",
  "authorization_endpoint":  "https://login.example.com/authorize",
  "token_endpoint":          "https://login.example.com/token",
  "registration_endpoint":   "https://login.example.com/register",
  "jwks_uri":                "https://login.example.com/jwks",
  "code_challenge_methods_supported": ["S256"]
}
```

### Registration

```http
POST /register HTTP/1.1
Host: login.example.com
Content-Type: application/json

{
  "client_name":                "Claude Desktop",
  "redirect_uris":              ["http://127.0.0.1:51247/cb"],
  "grant_types":                ["authorization_code", "refresh_token"],
  "response_types":             ["code"],
  "token_endpoint_auth_method": "none"
}
```

```http
HTTP/1.1 201 Created
{ "client_id": "mcp-cli-abc123", "client_id_issued_at": 1748352000 }
```

### Authorization request (browser)

```
https://login.example.com/authorize?
    response_type=code
    &client_id=mcp-cli-abc123
    &redirect_uri=http%3A%2F%2F127.0.0.1%3A51247%2Fcb
    &scope=mcp%3Atools.invoke
    &state=xyz
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
    &resource=https%3A%2F%2Fmcp.example.com
```

### Token request

```http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=http%3A%2F%2F127.0.0.1%3A51247%2Fcb
&client_id=mcp-cli-abc123
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
&resource=https%3A%2F%2Fmcp.example.com
```

```http
HTTP/1.1 200 OK
{
  "access_token":  "eyJhbGciOiJSUzI1NiIs...",
  "token_type":    "Bearer",
  "expires_in":    3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope":         "mcp:tools.invoke"
}
```

### MCP call

```http
POST /mcp HTTP/1.1
Host: mcp.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json

{"jsonrpc":"2.0","method":"tools/list","id":1}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"jsonrpc":"2.0","result":{"tools":[...]},"id":1}
```

## Once-only vs per-call

```mermaid
flowchart LR
    subgraph Once[Once per user / per AS / per server]
        D1[Discovery — PRM + AS metadata]
        R[Registration — DCR]
    end
    subgraph PerSession[Once per session]
        A[Authorization + token exchange]
    end
    subgraph PerCall[Every MCP request]
        T[Bearer token]
        V[Server-side validation]
    end
    Once --> PerSession --> PerCall
```

Most of the complexity happens once. After the user is signed in, the steady state is just "send bearer, validate token, respond."

---

← [Resource indicators](04-resource-indicators.md) · ↑ [MCP](README.md) · → Next: [Server implementation](06-server-implementation.md)
