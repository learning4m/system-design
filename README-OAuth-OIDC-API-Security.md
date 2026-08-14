# OAuth 2.0, OIDC, JWT & API Security --- Deep Understanding Guide

> This guide is intentionally tutorial-style. It explains **why** each
> mechanism exists, **what happens step-by-step**, and **how the
> concepts connect in Spring Security and microservices**.

## Contents

1.  [Build the security mental model
    first](#1-build-the-security-mental-model-first)
2.  [TLS and mTLS](#2-tls-and-mtls)
3.  [mTLS when TLS terminates at
    Ingress](#3-mtls-when-tls-terminates-at-ingress)
4.  [API keys and request signing](#4-api-keys-and-request-signing)
5.  [Session/cookie authentication, HttpOnly and
    CORS](#5-sessioncookie-authentication-httponly-and-cors)
6.  [Why OAuth exists](#6-why-oauth-exists)
7.  [OAuth actors](#7-oauth-actors)
8.  [OAuth vs OIDC](#8-oauth-vs-oidc)
9.  [Access Token vs ID Token vs Refresh
    Token](#9-access-token-vs-id-token-vs-refresh-token)
10. [Authorization Code + PKCE](#10-authorization-code--pkce)
11. [`state` vs `nonce` vs PKCE](#11-state-vs-nonce-vs-pkce)
12. [Client Credentials](#12-client-credentials)
13. [What a Resource Server actually
    does](#13-what-a-resource-server-actually-does)
14. [JWT validation --- every step](#14-jwt-validation--every-step)
15. [JWKS, `kid`, caching and key
    rotation](#15-jwks-kid-caching-and-key-rotation)
16. [Does JWT validation happen on every
    request?](#16-does-jwt-validation-happen-on-every-request)
17. [Opaque-token validation](#17-opaque-token-validation)
18. [JWT vs opaque --- the real
    trade-off](#18-jwt-vs-opaque--the-real-trade-off)
19. [Spring Security internals](#19-spring-security-internals)
20. [Scopes, roles, permissions and
    claims](#20-scopes-roles-permissions-and-claims)
21. [RBAC, ABAC and object-level
    authorization](#21-rbac-abac-and-object-level-authorization)
22. [Token propagation across
    microservices](#22-token-propagation-across-microservices)
23. [Logout and token/session
    lifecycle](#23-logout-and-tokensession-lifecycle)
24. [Complete architecture](#24-complete-architecture)
25. [Interview-ready mental models](#25-interview-ready-mental-models)

------------------------------------------------------------------------

# 1. Build the security mental model first

Do not begin by memorizing OAuth terms. First separate the security
questions.

``` text
React UI → Ingress/Gateway → Order Service → Payment Service
```

The system must answer different questions:

``` text
Can somebody read/modify traffic?
        ↓
      TLS

Which machine/service established this connection?
        ↓
      mTLS

Who is the human user?
        ↓
Session / OIDC

Is this client allowed to call this API?
        ↓
OAuth Access Token / API Key

What API capability was granted?
        ↓
OAuth Scope

May this caller operate on THIS account/payment?
        ↓
Business authorization: RBAC / ABAC / ownership
```

These are **layers**, not competing alternatives.

A cryptographically valid JWT does not automatically mean that Alice can
access account `12345`. It only gets you through part of the security
decision.

------------------------------------------------------------------------

# 2. TLS and mTLS

## TLS

With normal HTTPS:

``` text
Client                                  Server
  │                                       │
  │──── ClientHello ─────────────────────►│
  │◄── server certificate + handshake ────│
  │                                       │
  │ verifies server identity              │
  │                                       │
  │════ encrypted HTTP communication ═════│
```

TLS primarily gives:

-   confidentiality;
-   integrity of network traffic;
-   server authentication.

User authentication happens at another layer.

## mTLS

mTLS adds client-certificate authentication:

``` text
Order Service                          Payment Service
     │                                       │
     │◄──── server certificate ───────────────│
     │──── client certificate ───────────────►│
     │                                       │
     │       both sides authenticate         │
     │══════ encrypted connection ═══════════│
```

Payment Service now knows that the TLS peer possessed the private key
corresponding to a trusted client certificate.

### Java servlet case

If TLS terminates directly at Tomcat/Jetty, the servlet container
participated in the TLS handshake and can expose the validated
certificate chain:

``` java
X509Certificate[] certs =
    (X509Certificate[]) request.getAttribute(
        "jakarta.servlet.request.X509Certificate"
    );
```

The correct sequence is:

``` text
Certificate received
      ↓
TLS validates chain/trust
      ↓
Servlet exposes validated certificate
      ↓
Application reads CN/SAN/OU
      ↓
Application maps identity to authorization
```

Do not treat a `CN` string itself as proof of identity. The trust comes
from cryptographic certificate validation.

------------------------------------------------------------------------

# 3. mTLS when TLS terminates at Ingress

This changes the trust model substantially.

``` text
External Client
      │
      │ mTLS connection #1
      ▼
    Ingress
      │
      │ HTTP or TLS connection #2
      ▼
  Spring Boot
```

The original TLS session **ends at the Ingress**. Spring Boot did not
see the original TLS handshake.

Therefore the normal servlet X.509 attribute cannot magically contain
the original client certificate.

A common design is:

``` text
Client certificate
       ↓
Ingress
 ├─ verify trusted CA
 ├─ validate certificate
 ├─ derive identity from SAN/subject
 └─ create trusted internal identity header
       ↓
Spring Boot
```

For example:

``` http
X-Verified-Client-Subject: CN=order-service,OU=Payments
```

But this is safe only if:

``` text
Internet
   │
   X── cannot bypass Ingress
   │
Ingress
   │ strips/overwrites external X-Verified-* headers
   ▼
Backend
```

Otherwise an attacker could simply type the trusted header themselves.

With **TLS passthrough**, the original encrypted connection reaches the
application and Java can authenticate the certificate directly.

------------------------------------------------------------------------

# 4. API keys and request signing

## API key

``` text
Client
  │ X-API-Key: abc123
  ▼
API
  ├─ look up key
  ├─ identify application
  ├─ check active?
  └─ enforce quota/policy
```

An API key is useful for relatively simple application
identification/access.

It does not naturally model rich user delegation.

### API key is not an OAuth client secret

``` text
API KEY:
Client ───── API Key ─────► API


OAUTH CLIENT SECRET:
Client ── client secret ──► Authorization Server
                                │
                                ▼
                           Access Token
                                │
Client ───── Access Token ─────► API
```

## Request signing

With HMAC, client and server share the same secret:

``` text
Client                              Server
  │ Secret K                          │ Secret K
  │                                  │
  │ HMAC(K, canonical-request)       │
  │──── request + signature ─────────►│
  │                                  │ recompute
  │                                  │ compare
```

Yes, the secret must be securely provisioned to both sides beforehand.

Asymmetric signing instead uses:

``` text
Client Private Key → sign
Server Public Key  → verify
```

A robust signed request often includes method, URI, body digest,
timestamp and nonce/request ID to resist tampering and replay.

------------------------------------------------------------------------

# 5. Session/cookie authentication, HttpOnly and CORS

## Session authentication

``` text
Browser
   │ login credentials
   ▼
Spring
   ├─ authenticate
   ├─ create SecurityContext
   └─ store in HttpSession
          ↓
      session id ABC
          ↓
Set-Cookie: JSESSIONID=ABC
```

Next request:

``` http
Cookie: JSESSIONID=ABC
```

Spring restores:

``` text
ABC → HttpSession → SecurityContext → Authentication
```

React/Angular does not prevent this. Cookies are browser functionality.

## HttpOnly

``` http
Set-Cookie: JSESSIONID=ABC; HttpOnly; Secure
```

Means:

``` text
JavaScript reads cookie directly?  NO
Browser sends matching cookie?     YES
```

`HttpOnly` helps reduce cookie theft through injected JavaScript. It is
not a complete XSS or CSRF solution.

## CORS credentials

Frontend:

``` javascript
fetch("https://api.example.com/me", {
  credentials: "include"
});
```

This asks the browser to include applicable credentials such as cookies.

Server may respond:

``` http
Access-Control-Allow-Origin: https://ui.example.com
Access-Control-Allow-Credentials: true
```

A crucial distinction:

``` text
Request reaches API
      ↓
API may authenticate/process it
      ↓
Response reaches browser
      ↓
Browser applies CORS policy
      ↓
JavaScript may or may not be allowed to read response
```

Therefore, missing `Access-Control-Allow-Credentials: true` does **not**
mean the server necessarily refused to execute the request.

CORS is primarily a browser cross-origin response-sharing policy, **not
server authorization**.

## Preflight

For a non-simple cross-origin request, the browser itself sends
`OPTIONS` first:

``` text
JavaScript wants cross-origin request
          ↓
Browser checks method/headers/content-type
          ↓
Simple CORS request?
     ┌────┴────┐
    YES        NO
     │          │
actual       OPTIONS
request      preflight
```

------------------------------------------------------------------------

# 6. Why OAuth exists

Imagine a finance application wants access to Alice's bank transactions.

Bad design:

``` text
Alice gives bank username/password
to Finance App
      ↓
Finance App impersonates Alice
```

Problems: password sharing, excessive privilege, difficult revocation,
poor auditing.

OAuth changes the model:

``` text
Alice
  │ authorizes limited access
  ▼
Authorization Server
  │ issues limited Access Token
  ▼
Finance App
  │ token
  ▼
Bank API
```

Alice's password remains with the Authorization Server.

The Finance App gets a **delegated credential**, potentially limited by
resource, scope and lifetime.

That is the core idea:

> **OAuth is about delegated authorization.**

------------------------------------------------------------------------

# 7. OAuth actors

Concrete mapping:

``` text
Alice                         Resource Owner
Finance App                   Client
login.bank.com                Authorization Server
api.bank.com                  Resource Server
```

Flow:

``` text
Resource Owner
     ↓
   Client
     ↓ authorization
Authorization Server
     ↓ access token
   Client
     ↓ access token
Resource Server
```

The Resource Server normally does not issue the access token; it
**accepts and validates** it.

------------------------------------------------------------------------

# 8. OAuth vs OIDC

``` text
OAuth 2.0
   ↓
Authorization
   ↓
"What access was granted to the client?"


OpenID Connect
   ↓
Authentication layer using OAuth infrastructure
   ↓
"Who authenticated?"
```

OIDC adds identity semantics, most notably the **ID Token**.

------------------------------------------------------------------------

# 9. Access Token vs ID Token vs Refresh Token

## Access Token

``` text
Client ── Access Token ──► Resource Server
```

Used for API authorization.

## ID Token

``` text
Authorization Server ── ID Token ──► OIDC Client
```

Used by the client as an authentication result/identity assertion.

Do **not** use the ID Token as a substitute for the API access token.

## Refresh Token

``` text
Client ── Refresh Token ──► Authorization Server
                                  │
                                  ▼
                           New Access Token
```

A refresh token normally goes to the token endpoint, not to business
APIs.

Mental model:

``` text
ID Token      → CLIENT asks: who authenticated?
Access Token  → API asks: what access is granted?
Refresh Token → AUTH SERVER asks: may I issue fresh access?
```

------------------------------------------------------------------------

# 10. Authorization Code + PKCE

``` text
Browser/Client
      │ 1. authorization request
      ▼
Authorization Server
      │ 2. authenticate user
      │ 3. obtain authorization
      ▼
Browser/Client
      │ 4. authorization code
      ▼
Token Endpoint
      │ 5. code + PKCE verifier
      ▼
Client receives tokens
```

The authorization code is temporary and is exchanged at the token
endpoint.

## Why PKCE?

Client generates:

``` text
code_verifier = random secret

code_challenge =
BASE64URL(SHA256(code_verifier))
```

Authorization request sends the challenge.

Token request sends the verifier.

``` text
Attacker steals authorization code
              │
              ▼
Does attacker know verifier?
              │
             NO
              ↓
            REJECT
```

PKCE is therefore a **code-redemption binding mechanism**, not
encryption of the authorization code.

------------------------------------------------------------------------

# 11. `state` vs `nonce` vs PKCE

Keep these separate:

``` text
state
  → correlates authorization response to request
  → helps defend authorization-response CSRF/confusion

PKCE
  → proves code redeemer has original verifier
  → protects authorization-code redemption

nonce
  → binds OIDC authentication response/ID Token
    to the authentication request
```

One login can use all three because they solve different problems.

------------------------------------------------------------------------

# 12. Client Credentials

For machine-to-machine access:

``` text
Order Service
     │ client authentication
     │ grant_type=client_credentials
     ▼
Authorization Server
     │
     ▼
Access Token
     │
     ▼
Order Service
     │ Bearer token
     ▼
Payment Service
```

There is no human user in this flow.

The token represents authority granted to the client/workload.

Client authentication can use a client secret, `private_key_jwt`, mTLS,
or other supported mechanisms.

------------------------------------------------------------------------

# 13. What a Resource Server actually does

Suppose Payment API receives:

``` http
POST /payments
Authorization: Bearer <access-token>
```

It must answer:

``` text
Is token authentic/active?
        ↓
Is it expired?
        ↓
Do I trust its issuer?
        ↓
Was it intended for this API?
        ↓
Does it have required scope?
        ↓
Does business policy permit THIS operation?
```

The first several questions are token authentication/validation.

The last questions are authorization.

------------------------------------------------------------------------

# 14. JWT validation --- every step

JWT:

``` text
HEADER.PAYLOAD.SIGNATURE
```

Example:

``` json
{
  "alg": "RS256",
  "kid": "KEY-A"
}
```

Claims:

``` json
{
  "iss": "https://auth.company.com",
  "sub": "alice",
  "aud": "payment-api",
  "scope": "payments.read",
  "exp": 1786690000
}
```

Issuer signs with a **private key**:

``` text
header + payload
      ↓
Private Key
      ↓
signature
```

Resource Server verifies with the corresponding **public key**.

Full validation:

``` text
Bearer JWT
   ↓
Parse header
   ↓
kid = KEY-A
   ↓
Find trusted Public Key A
   ↓
Verify signature
   ↓
Validate iss
   ↓
Validate exp/nbf
   ↓
Validate configured aud
   ↓
Run custom token validators
   ↓
Convert scopes/claims to authorities
   ↓
Authentication established
```

### Signature is necessary, but not sufficient

A valid signature means the signed data has not been modified and was
signed by the holder of the corresponding trusted private key.

You must still ask:

``` text
Trusted issuer?
Not expired?
Correct audience?
Required scope?
Correct tenant?
Allowed business object?
```

------------------------------------------------------------------------

# 15. JWKS, `kid`, caching and key rotation

Authorization Server:

``` text
Private Key A → signs
Public Key A  → published
```

JWKS conceptually contains:

``` text
KEY-A → Public Key A
KEY-B → Public Key B
```

JWT header contains:

``` text
kid = KEY-A
```

Resource Server does:

``` text
kid=KEY-A
    ↓
JWK cache
    ↓
Public Key A
    ↓
verify signature
```

## Rotation

Initially:

``` text
A → Public A
```

After rotation:

``` text
A → Public A   (old, temporarily retained)
B → Public B   (new)
```

New JWTs use `kid=B`.

A remains during the overlap so unexpired tokens signed with A can still
be validated.

When old A-signed tokens can no longer be valid, A can be removed.

------------------------------------------------------------------------

# 16. Does JWT validation happen on every request?

**Yes, in a stateless bearer-token Resource Server.**

``` text
Request #1 + JWT-X
      ↓
validate JWT-X

Request #2 + JWT-X
      ↓
validate JWT-X again
```

But this does **not** mean:

``` text
call Authorization Server every request
```

Normal path:

``` text
Request
  │ JWT, kid=A
  ▼
Resource Server
  │
  ▼
cached Public Key A
  │
  ▼
verify JWT locally
```

The thing normally cached is **public verification key material**, not a
permanent "this JWT is valid" decision.

Remember:

``` text
JWT validation every bearer request       YES

JWKS network call every bearer request     NO
```

If a new `kid=B` appears and B is absent from the local JWK cache, the
key source can refresh/retrieve JWKS.

------------------------------------------------------------------------

# 17. Opaque-token validation

Opaque token:

``` text
7b7c13a9e6f...
```

The Resource Server treats it as a reference/uninterpreted credential.

``` text
Client
  │ opaque token
  ▼
Resource Server
  │ introspection request
  ▼
Authorization Server
  │
  ├─ token exists?
  ├─ active?
  ├─ revoked?
  ├─ expired?
  ├─ subject?
  └─ scopes?
  ▼
Resource Server
```

Conceptual response:

``` json
{
  "active": true,
  "sub": "alice",
  "scope": "payments.read"
}
```

In the normal introspection model, this can happen on **every bearer
request**.

Caching is possible, but then central revocation is not visible until
the cached result expires.

------------------------------------------------------------------------

# 18. JWT vs opaque --- the real trade-off

  Question                         JWT                Opaque
  -------------------------------- ------------------ -------------------------------
  Validate every bearer request?   Yes                Yes
  Where?                           Normally local     Normally Authorization Server
  Network call each request?       No                 Usually yes
  Cached item                      Public keys/JWKS   Optional introspection result
  Immediate central revocation     Harder             Easier
  Runtime issuer dependency        Lower              Higher
  Horizontal scaling               Excellent          Introspection must scale

JWT:

``` text
Client → Resource Server
             │
             ├─ cached public key
             ├─ signature
             └─ claims
```

Opaque:

``` text
Client → Resource Server → Authorization Server
                               │
                               └─ active?
```

### Revocation example

``` text
10:00 JWT issued, expires 11:00
10:15 user disabled
10:30 JWT reaches API
```

Local validation may still show:

``` text
signature ✓
issuer    ✓
audience  ✓
expiry    ✓
```

The API may accept it because it did not ask the issuer whether central
user state changed.

This is why JWT systems commonly use short-lived access tokens and
separate refresh-token/session revocation strategies.

------------------------------------------------------------------------

# 19. Spring Security internals

## JWT

``` text
HTTP Request
    ↓
BearerTokenAuthenticationFilter
    ↓
BearerTokenAuthenticationToken
    ↓
AuthenticationManager
    ↓
JwtAuthenticationProvider
    ↓
JwtDecoder
    ├─ signature verification
    └─ claim validators
    ↓
JwtAuthenticationConverter
    ↓
JwtAuthenticationToken
    ↓
SecurityContext
    ↓
Authorization
```

## Opaque

``` text
HTTP Request
    ↓
BearerTokenAuthenticationFilter
    ↓
AuthenticationManager
    ↓
OpaqueTokenAuthenticationProvider
    ↓
OpaqueTokenIntrospector
    ↓
Authorization Server
    ↓
Authenticated principal + authorities
```

Also keep these Spring roles distinct:

``` text
oauth2Login()
  → log a user into this Spring application

oauth2Client()
  → use OAuth for outbound API calls

oauth2ResourceServer()
  → validate incoming Access Tokens
```

------------------------------------------------------------------------

# 20. Scopes, roles, permissions and claims

## Claim

Any asserted token field:

``` text
sub
aud
scope
tenant
department
```

## Scope

OAuth delegated API capability:

``` text
accounts.read
payments.create
```

## Role

Business/security classification:

``` text
CUSTOMER
TELLER
BRANCH_MANAGER
ADMIN
```

## Permission

Fine-grained capability:

``` text
PAYMENT_APPROVE
ACCOUNT_VIEW
```

A role may map to multiple permissions.

## Spring `GrantedAuthority`

Spring can represent authorization information as:

``` text
SCOPE_payments.read
ROLE_ADMIN
PAYMENT_APPROVE
```

A scope such as:

``` text
payments.read
```

commonly becomes:

``` text
SCOPE_payments.read
```

and can be checked with:

``` java
@PreAuthorize("hasAuthority('SCOPE_payments.read')")
```

`hasRole("ADMIN")` conventionally checks for `ROLE_ADMIN`.

------------------------------------------------------------------------

# 21. RBAC, ABAC and object-level authorization

Suppose Alice has:

``` text
SCOPE_accounts.read
```

Request:

``` http
GET /accounts/12345
```

The scope does **not** automatically mean Alice can read every account.

Think in layers:

``` text
Token valid?
   ↓
accounts.read scope?
   ↓
Is account 12345 accessible to Alice?
   ↓
Correct tenant?
   ↓
Other business restrictions?
   ↓
ALLOW
```

## RBAC

``` text
User → Role → Permissions
```

## ABAC

Uses attributes and context:

``` text
user.branch == payment.branch
AND
payment.amount <= user.approvalLimit
```

OAuth scope is usually coarse-grained. Domain/object authorization
finishes the decision.

------------------------------------------------------------------------

# 22. Token propagation across microservices

Suppose:

``` text
Alice
  │ T1
  ▼
Service A
  │
  ▼
Service B
```

Do not begin with:

> "Should A forward T1?"

Begin with:

> **Whose authority should B receive?**

## Token relay

``` text
Alice ── T1 ──► A ── T1 ──► B
```

Only appropriate if T1 is legitimately intended for B.

If:

``` text
aud = service-a
```

B should not accept it merely because the signature is valid.

## Client Credentials

If B only needs Service A's identity/authority:

``` text
A → Authorization Server → service token → B
```

## Token Exchange / OBO

If B needs Alice's delegated context but should receive a B-specific
token:

``` text
Alice
 │ T1, aud=A
 ▼
A
 │ exchange
 ▼
Authorization Server
 │
 ▼
T2, aud=B
 │
 ▼
B
```

## mTLS + OAuth

These can complement each other:

``` text
mTLS
  → Which workload called me?

OAuth token
  → What user/service authority is being exercised?
```

------------------------------------------------------------------------

# 23. Logout and token/session lifecycle

A user may simultaneously have:

``` text
Spring HttpSession
IdP SSO session
Access Token
Refresh Token
```

These have independent lifetimes.

## Local logout

``` text
POST /logout
   ↓
clear SecurityContext
   ↓
invalidate HttpSession
   ↓
remove JSESSIONID
```

This does not necessarily terminate the IdP SSO session.

So the user may click Login again and be immediately reauthenticated
because the IdP still has an SSO cookie/session.

## JWT after logout

``` text
JWT expires at 11:00
local logout at 10:30
```

A copied JWT may remain valid until 11:00.

Therefore:

``` text
Local logout
   ≠ Access-token revocation
   ≠ Refresh-token revocation
   ≠ IdP logout
```

This is why token lifetime and revocation strategy matter.

------------------------------------------------------------------------

# 24. Complete architecture

``` text
Browser / Client
      │
      │ HTTPS
      ▼
Ingress / Gateway
      │
      │ optional mTLS identity
      ▼
Spring Resource Server
      │
      │ Bearer Token
      ▼
Token Authentication
   ┌──┴──────────────┐
   │                 │
  JWT              Opaque
   │                 │
local validation   introspection
   │                 │
cached JWKS       Auth Server
   └───────┬─────────┘
           ▼
     Authentication
           │
           ▼
     OAuth scopes
           │
           ▼
 Roles / permissions
           │
           ▼
Object/domain authorization
           │
           ▼
     Business logic
           │
           ▼
   Downstream service
      ┌────┼─────────┐
      │    │         │
    relay client   token
          creds   exchange
```

The clean architecture is:

``` text
Transport Security
       ↓
Authentication
       ↓
Token Validation
       ↓
OAuth Authorization
       ↓
Business Authorization
       ↓
Downstream Delegation
```

------------------------------------------------------------------------

# 25. Interview-ready mental models

## How does JWT validation work?

> A Resource Server extracts the bearer JWT, selects a trusted public
> key---commonly from cached JWKS using the token's `kid`---verifies the
> signature, validates issuer and timestamps plus configured
> audience/policy, converts trusted scopes/claims to authorities, and
> establishes authentication. This occurs on every stateless
> bearer-token request, but normally without an Authorization Server
> call on each request because the verification keys are cached.

## How does opaque validation work?

> The Resource Server treats the token as a reference credential and
> calls the Authorization Server's introspection endpoint to determine
> whether it is active and obtain metadata such as subject, scope and
> expiry. Normally this happens per bearer request unless an explicit
> cache is introduced.

## JWT vs opaque?

``` text
JWT
  → decentralized/local validation
  → lower runtime coupling
  → excellent scalability
  → harder immediate revocation

Opaque
  → centralized introspection
  → stronger immediate central control
  → network/runtime dependency
```

## mTLS vs OAuth?

Not necessarily alternatives:

``` text
mTLS  → authenticate workload/connection
OAuth → authorize API access/delegation
```

## Is a valid JWT enough?

No:

``` text
Signature valid?
      ↓
Trusted issuer?
      ↓
Correct audience?
      ↓
Not expired?
      ↓
Required scope?
      ↓
Can this principal access THIS resource?
      ↓
Business policy?
      ↓
ALLOW
```

------------------------------------------------------------------------

# Final Memory Map

``` text
TLS
 └─ secure channel

mTLS
 └─ secure channel + client/workload authentication

Session
 └─ browser holds session ID; server holds auth state

OAuth 2.0
 └─ delegated API authorization
    ├─ Authorization Code + PKCE
    └─ Client Credentials

OIDC
 └─ authentication/identity layer
    └─ ID Token

Access Token
 └─ goes to Resource Server
    ├─ JWT → local signature/claim validation
    └─ Opaque → introspection

Spring Resource Server
 ├─ BearerTokenAuthenticationFilter
 ├─ JwtAuthenticationProvider / JwtDecoder
 └─ OpaqueTokenAuthenticationProvider / Introspector

Authorization
 ├─ scopes
 ├─ roles
 ├─ permissions
 ├─ RBAC
 ├─ ABAC
 └─ object/business rules

Microservices
 ├─ token relay
 ├─ client credentials
 ├─ token exchange/OBO
 └─ mTLS/workload identity

Logout
 ├─ local application session
 ├─ IdP SSO session
 ├─ access token
 └─ refresh token
    → separate lifecycles
```

## Ten rules worth retaining

1.  **TLS protects the channel; it does not replace application
    authorization.**
2.  **mTLS can identify a workload; OAuth can independently authorize
    API access.**
3.  **OAuth is primarily delegated authorization; OIDC adds
    authentication.**
4.  **Access Tokens are for Resource Servers; ID Tokens are for OIDC
    clients.**
5.  **PKCE, `state` and `nonce` solve different problems.**
6.  **JWTs are validated on every stateless bearer request, but JWKS is
    not normally fetched every time.**
7.  **Opaque tokens normally require introspection for every bearer
    request unless caching is deliberately added.**
8.  **A valid access token does not automatically authorize every
    business object.**
9.  **Do not blindly forward one JWT through all microservices; first
    decide whose authority the downstream call should exercise.**
10. **Logout, token expiry, token revocation and IdP SSO termination are
    different lifecycle events.**
