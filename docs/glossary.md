# Glossary — plain-language definitions

> **In one line:** Every important word in this guide, explained in one or two everyday sentences.
>
> **Why it matters:** You can keep this page open while reading anything else. When a term stops making sense, look it up here instead of flipping back through chapters.

Terms are grouped by theme rather than alphabetically, so related ideas sit together. Where a fuller explanation exists, the chapter is linked.

## The big picture

**OAuth**
A system that lets an app use part of your account on another service *without* you giving the app your password. Instead of your password, the app receives a limited, temporary pass. See [Chapter 1](01-what-is-oauth.md).

**Authentication**
Proving *who you are* — like showing ID at a door. ("This really is Philippe.")

**Authorization**
Deciding *what you are allowed to do* once you're known — like a ticket that opens some doors but not others. ("Philippe may read mail, but not delete it.") OAuth is mostly about this part.

**OpenID Connect**
A small add-on to OAuth that also tells the app *who the person is*, not just what they may do. It's the technology behind "Sign in with Google / Apple." See [Chapter 8](08-oidc.md).

## The players

**Resource owner (the user)**
The person who owns the data and is granting access — usually you.

**Client (the app)**
The application that wants access — a website, a phone app, a desktop tool, an AI assistant. Note: the *client* is the app, not the person using it.

**Authorization server**
The trusted gatekeeper that logs the user in, asks their permission, and hands out passes. Examples: Google, Okta, Microsoft Entra, Auth0, Keycloak.

**Resource server (the API)**
The service holding the actual data the app wants to reach — for example, the mail service. Its only job in this system is to check each pass and then answer or refuse.

**Identity provider**
A common everyday name for the gatekeeper that logs people in. In practice it's usually the same thing as the authorization server.

## Passes and permissions

**Token**
The general word for the temporary pass the app receives. The app shows it to a service to get in. Anyone reading this guide will see "token" constantly — it just means "access pass." See [Chapter 5](05-tokens.md).

**Access token**
The short-lived pass the app actually uses on each request to the service. Usually lasts minutes to a few hours.

**Refresh token**
A longer-lived pass whose only purpose is to quietly get a new access token when the old one expires — so the user isn't asked to log in again every hour. See [Refresh Token](flows/refresh-token.md).

**ID token**
A small signed note that tells the app *who the user is* (name, a stable user number, email). Comes from OpenID Connect. The app reads it; it is never shown to the data service.

**Scope**
The list of specific things a pass is allowed to do, like "read mail" or "see calendar." The user approves these on the consent screen. See the scopes section in [Chapter 2](02-concepts-vocabulary.md).

**Consent**
The screen where the user is asked "App X wants to read your mail — allow?" and clicks yes or no.

**Audience**
The single service a pass is meant for. A pass stamped for the mail service should be refused everywhere else. See [Resource indicators](mcp/04-resource-indicators.md).

**Issuer**
Who handed out the pass — the gatekeeper's identity. A service checks the issuer to make sure a pass came from a gatekeeper it trusts.

**Claim**
Any single fact written inside a token — the user's number, the audience, the expiry time, the allowed scopes. A token is basically a bundle of claims.

## How passes are built and checked

**Bearer token**
The plain kind of pass: whoever holds it can use it, like cash. Simple, but if it's stolen it works for the thief too. See [Chapter 5](05-tokens.md).

**Sender-constrained token**
A stronger pass that is tied to the specific app it was given to, so a stolen copy is useless to anyone else. Two common forms are **mutual TLS** and **DPoP** (below).

**mutual TLS (mTLS)**
A method where both sides of a connection prove their identity with certificates. Used to lock a pass to one app.

**DPoP**
A lighter-weight way to lock a pass to one app: the app signs a tiny fresh proof on every request using a private key only it holds. A stolen pass without that key does nothing. See [Beyond bearer](mcp/08-beyond-bearer.md).

**JWT**
A common token format: a signed bundle of facts that a service can check on its own without phoning the gatekeeper. Pronounced "jot." 

**JWKS**
The set of public keys a gatekeeper publishes so services can verify that a signed token is genuine. Think of it as the gatekeeper's published signature samples.

**PKCE**
A safeguard, used during browser logins, that ties the login to the exact app that started it — so an attacker who intercepts the handover partway through still can't finish it and steal access. Pronounced "pixy." See [Authorization Code + PKCE](flows/authorization-code-pkce.md).

**Redirect (redirect URI)**
The exact web address the gatekeeper sends the user back to after they log in. It must be registered in advance so access can't be diverted elsewhere.

## Discovery and registration

**Discovery**
How an app, starting from just a web address, automatically learns where to log in and what it can ask for — instead of having all that wired in by hand. See [Discovery chain](mcp/02-discovery-chain.md).

**Dynamic client registration**
A way for a brand-new app to introduce itself to a gatekeeper automatically, with no human setting it up first. See [Dynamic Client Registration](mcp/03-dynamic-client-registration.md).

## The AI / agent side

**MCP (Model Context Protocol)**
An open standard for how AI assistants connect to outside tools and data sources. See [Chapter 10](mcp/README.md).

**MCP host**
The application the AI runs inside — for example, Claude Desktop. It's the part that actually holds the passes and talks to the gatekeeper.

**MCP client**
The piece inside the host that maintains the connection to one specific tool.

**MCP server**
A tool or data source exposed to the AI — a mail tool, a calendar tool, a file tool. In this system its only security job is to check passes.

**Agent**
The AI doing the reasoning and deciding which tools to use. Importantly, the agent is *not* the thing that holds the passes — the host is. See the [Agent / MCP pattern](mcp/09-agent-pattern-end-to-end.md).

**Workload identity**
A way for machines and automated programs (not people) to prove who they are without being handed a permanent password. See [Chapter 9](09-workload-identity.md).

**Token exchange**
Trading a broad pass for a narrower one aimed at a single downstream service — so each step gets only the access it needs. See [Token Exchange](flows/token-exchange.md).

**`act` claim**
A note inside a token recording that an agent is acting on a user's behalf, so audit logs can tell "the user did this" apart from "the agent did this for the user."

---

← [README](../README.md)
