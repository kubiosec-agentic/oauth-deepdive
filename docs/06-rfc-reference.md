# 6. The OAuth ecosystem — supporting RFCs you should know

> **In one line:** A quick lookup table of the official documents behind each feature mentioned in this guide.
>
> **Why it matters:** When you need the authoritative source for a detail, this tells you exactly which document to open.

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

If you have one slot in your reading list after RFC 6749, make it **RFC 9700**: it's the security state of the art compressed into one document, and it's what OAuth 2.1 is built from.

For OIDC, the canonical specs are not IETF RFCs but OpenID Foundation documents: see the [OIDC chapter](08-oidc.md).

---

← [Tokens](05-tokens.md) · ↑ [README](../README.md) · → Next: [OAuth 2.1](07-oauth-2.1.md)
