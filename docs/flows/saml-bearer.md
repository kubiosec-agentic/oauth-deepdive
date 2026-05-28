# 4.8 SAML 2.0 Bearer assertion grant (RFC 7522)

> **In one line:** The same idea as the previous page, but using an older corporate sign-in format.
>
> **Why it matters:** You will mostly meet this when bridging older company systems to newer ones. For anything new, the previous page is the better choice.

The same pattern as [JWT Bearer](jwt-bearer.md), but the assertion is a SAML 2.0 token instead of a JWT.

## When you'll see this

- Enterprise bridges between legacy SAML SSO IdPs and modern OAuth-based APIs.
- Tenant migrations where the SAML investment is sunk and OAuth-side APIs are new.

New systems should prefer JWT bearer — SAML XML signatures are an attack surface (XML signature wrapping has bitten many implementations historically).

## HTTP

```http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:saml2-bearer
&assertion=<base64url-encoded SAML assertion>
```

## Practical guidance

If you're not maintaining an existing SAML deployment, you almost certainly want JWT Bearer instead.

---

← [JWT Bearer](jwt-bearer.md) · ↑ [Flows](README.md) · → Next: [Token Exchange](token-exchange.md)
