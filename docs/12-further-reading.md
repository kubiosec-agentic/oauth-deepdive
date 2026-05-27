# 11. Further reading

## Specs

Chronological by relevance.

- **RFC 6749** — OAuth 2.0 Authorization Framework
- **RFC 6750** — Bearer Token Usage
- **RFC 7636** — PKCE
- **RFC 7591** — Dynamic Client Registration
- **RFC 8252** — OAuth 2.0 for Native Apps (BCP 212)
- **RFC 8414** — Authorization Server Metadata
- **RFC 8628** — Device Authorization Grant
- **RFC 8693** — Token Exchange
- **RFC 8707** — Resource Indicators
- **RFC 9068** — JWT Profile for Access Tokens
- **RFC 9449** — DPoP
- **RFC 9700** — OAuth 2.0 Security Best Current Practice (BCP)
- **RFC 9728** — Protected Resource Metadata
- **draft-ietf-oauth-v2-1** — OAuth 2.1 (current draft -15, March 2026)

## OIDC

- **OpenID Connect Core 1.0** — the canonical OIDC spec
- **OpenID Connect Discovery 1.0** — `/.well-known/openid-configuration`
- **OpenID Connect Dynamic Client Registration 1.0** — predecessor and superset of RFC 7591
- **FAPI 2.0 Security Profile** — high-assurance OIDC + OAuth 2.1

## MCP authorization

- [MCP specification — Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization) — the canonical spec
- [The 2026-07-28 MCP Specification Release Candidate](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/) — what's coming
- [MCP, OAuth 2.1, PKCE, and the Future of AI Authorization (Aembit)](https://aembit.io/blog/mcp-oauth-2-1-pkce-and-the-future-of-ai-authorization/) — accessible overview
- [Technical Deconstruction of MCP Authorization (kane.mx)](https://kane.mx/posts/2025/mcp-authorization-oauth-rfc-deep-dive/) — RFC-by-RFC walkthrough

## Books and long-form

- *OAuth 2 in Action* — Justin Richer and Antonio Sanso. Still the best book on the framework.
- [oauth.net](https://oauth.net/2.1/) — Aaron Parecki's curated reference, kept current.
- [oauth.com](https://oauth.com/) — Aaron Parecki's tutorial-style site.

## If you want to go deeper

- **The OAuth Working Group archives** at IETF — the discussions around 2.1 are educational.
- **The MCP working group on GitHub** — `modelcontextprotocol/modelcontextprotocol` issues — open conversations about pending profile decisions.
- **Critical reviews:** Eran Hammer's blog posts from the 2.0 era (still surprisingly relevant), and the academic literature on OAuth attacks (search for "Daniel Fett" and "Ralf Küsters" — formal analyses of the protocol).

---

← [Security](11-security.md) · ↑ [README](../README.md)

*If you'd like a follow-up: a working MCP authorization implementation walkthrough (server-side code in your language of choice, full discovery + token-validation flow), a security review of an existing OAuth deployment, or a deeper dive on agent-specific patterns like Token Exchange fan-out — just ask.*
