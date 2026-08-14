# OAuth 2.0, OIDC & API Security --- Practical Reference

> A structured reference covering API security, OAuth 2.0, OpenID
> Connect, JWT/opaque tokens, Spring Security internals, microservice
> token propagation, logout, and authorization design.

## Table of Contents

1.  [Security Mechanisms](#1-security-mechanisms)
2.  [API Keys](#2-api-keys)
3.  [TLS and mTLS](#3-tls-and-mtls)
4.  [mTLS at Ingress](#4-mtls-at-ingress)
5.  [Request Signing](#5-request-signing)
6.  [Sessions, Cookies and CORS](#6-sessions-cookies-and-cors)
7.  [OAuth vs OIDC](#7-oauth-vs-oidc)
8.  [OAuth Tokens and Actors](#8-oauth-tokens-and-actors)
9.  [Authorization Code + PKCE](#9-authorization-code--pkce)
10. [Client Credentials](#10-client-credentials)
11. [OAuth Security Defenses](#11-oauth-security-defenses)
12. [Spring Security Internals](#12-spring-security-internals)
13. [JWT Validation](#13-jwt-validation)
14. [Opaque Token Validation](#14-opaque-token-validation)
15. [JWT vs Opaque](#15-jwt-vs-opaque)
16. [Microservice Token Propagation](#16-microservice-token-propagation)
17. [Logout and Session Lifecycle](#17-logout-and-session-lifecycle)
18. [Scopes, Roles, Permissions and
    Claims](#18-scopes-roles-permissions-and-claims)
19. [RBAC, ABAC and Object
    Authorization](#19-rbac-abac-and-object-authorization)
20. [Architect Decision Guide](#20-architect-decision-guide)
21. [Quick Revision](#21-quick-revision)

------------------------------------------------------------------------

## 1. Security Mechanisms

  -------------------------------------------------------------------------------
  Mechanism               Purpose                         Typical use
  ----------------------- ------------------------------- -----------------------
  TLS/HTTPS               Encryption + server             Almost all HTTP APIs
                          authentication                  

  mTLS                    Mutual client/server            Service-to-service, B2B
                          authentication                  

  API Key                 Application                     Simple integrations
                          identification/authentication   

  Session + Cookie        Stateful browser authentication Web apps/BFF

  OAuth 2.0               Delegated API authorization     Modern APIs

  OIDC                    User authentication on OAuth    Login/SSO

  JWT access token        Self-contained access           Distributed APIs
                          credential                      

  Opaque token            Centrally validated reference   Central control
                          credential                      

  Request signing         Request authenticity/integrity  Financial/cloud APIs
  -------------------------------------------------------------------------------

``` text
TLS / mTLS
    ↓
Transport and workload authentication

OAuth Access Token
    ↓
API authorization

OIDC ID Token
    ↓
User authentication result

Application authorization
    ↓
Roles / permissions / ownership / business rules
```

------------------------------------------------------------------------

## 2. API Keys

An API key is a credential issued to an application.

``` text
Client
  │
  │ X-API-Key: abc123
  ▼
API
  └── validate key
```

An API key is normally presented directly to the API.

An OAuth client secret is different:

``` text
Client ── client_secret ──► Authorization Server
                                │
                                ▼
                           Access Token
                                │
Client ── Access Token ────────► API
```

The downstream API normally never needs the OAuth client's secret.

------------------------------------------------------------------------

## 3. TLS and mTLS

### Normal TLS

``` text
Client
   │ verifies server certificate
   ▼
Server
```

### Mutual TLS

``` text
Client                               Server
  │◄──── server certificate ──────────│
  │──── client certificate ──────────►│
  └──── encrypted TLS channel ────────┘
```

With mTLS, both sides authenticate cryptographically.

When TLS terminates directly at the servlet container, Java can access
the validated client certificate chain through the servlet request's
X.509 certificate attribute.

Conceptually:

``` java
X509Certificate[] certs =
    (X509Certificate[]) request.getAttribute(
        "jakarta.servlet.request.X509Certificate"
    );
```

The certificate's CN/OU/SAN values are useful only after the TLS layer
has validated the certificate and its trust chain.

------------------------------------------------------------------------

## 4. mTLS at Ingress

If TLS terminates at an Ingress:

``` text
Client
  │ mTLS
  ▼
Ingress
  │ new HTTP/TLS hop
  ▼
Spring Boot
```

Spring Boot did not participate in the original client TLS handshake.

Therefore the original client certificate is not automatically available
through the normal servlet X.509 attribute.

A common design is:

``` text
Client Certificate
       │
       ▼
Ingress
  ├─ validate certificate
  ├─ validate CA/trust
  └─ derive identity
       │
       ▼
Trusted identity header
       │
       ▼
Spring Boot
```

Security requirement: external callers must not be able to bypass the
Ingress and spoof that trusted header. The Ingress should
strip/overwrite externally supplied identity headers.

With TLS passthrough, TLS instead terminates at the application,
allowing Java to authenticate the client certificate directly.

------------------------------------------------------------------------

## 5. Request Signing

With symmetric HMAC signing:

``` text
Client                         Server
  │ Shared Secret K              │ Shared Secret K
  │                              │
  │ HMAC(K, request data)        │
  │──── request + signature ────►│
  │                              │ recompute + compare
```

Yes, both sides must securely receive the same secret beforehand.

With asymmetric signing:

``` text
Client Private Key
       │
       └── sign request

Server Public Key
       │
       └── verify signature
```

Strong request-signing designs include request-specific values such as
method, URI, body digest, timestamp and nonce to reduce tampering and
replay.

------------------------------------------------------------------------

## 6. Sessions, Cookies and CORS

Traditional session authentication:

``` text
Browser
   │ login
   ▼
Server
   │ authenticate
   │ create session
   ▼
JSESSIONID=ABC
```

Later:

``` http
GET /account
Cookie: JSESSIONID=ABC
```

React or Angular does not prevent browser cookies from working.

### HttpOnly

``` http
Set-Cookie: JSESSIONID=ABC; HttpOnly; Secure
```

``` text
JavaScript reads HttpOnly cookie?   NO
Browser sends matching cookie?      YES
```

`HttpOnly` reduces cookie theft through JavaScript/XSS. It does not by
itself solve CSRF.

### Secure

`Secure` means the cookie should be sent over HTTPS.

### SameSite

Controls cross-site cookie behavior:

``` text
Strict
Lax
None
```

### Cross-origin credentials

A frontend may use:

``` javascript
fetch("https://api.example.com/me", {
    credentials: "include"
});
```

The server may need:

``` http
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Credentials: true
```

Important:

> CORS controls what cross-origin responses browser JavaScript may
> access. It is not a substitute for server-side authentication or
> authorization.

A server can process a request even though the browser later refuses to
expose its response to JavaScript.

### Preflight

``` text
Cross-origin request
       │
       ▼
CORS simple request?
    /       \
  YES       NO
   │         │
 send      OPTIONS
 request    preflight
             │
             ▼
        allowed?
```

------------------------------------------------------------------------

## 7. OAuth vs OIDC

``` text
OAuth 2.0
    ↓
Authorization
    ↓
Can this client access an API?


OIDC
    ↓
Authentication layer on OAuth
    ↓
Who is the user?
```

### Access Token vs ID Token

``` text
Access Token
     ↓
Resource Server/API


ID Token
     ↓
OIDC Client
```

Do not use an ID Token as an API access token.

------------------------------------------------------------------------

## 8. OAuth Tokens and Actors

Main actors:

``` text
Resource Owner
    usually user

Client
    requests access

Authorization Server
    issues tokens

Resource Server
    accepts access tokens
```

### Access Token

``` http
Authorization: Bearer <access-token>
```

Used by an API for authorization.

### ID Token

OIDC authentication result intended for the client application.

### Refresh Token

Used to obtain new access tokens without repeating the full interactive
login.

------------------------------------------------------------------------

## 9. Authorization Code + PKCE

``` text
Client
  │ authorization request
  ▼
Authorization Server
  │ user authentication
  ▼
Authorization Code
  │
  ▼
Client
  │ code + code_verifier
  ▼
Token Endpoint
  │
  ├─ Access Token
  ├─ ID Token (OIDC)
  └─ Refresh Token (possibly)
```

PKCE:

``` text
code_verifier = random secret

code_challenge =
BASE64URL(SHA256(code_verifier))
```

The challenge goes with the authorization request. The verifier goes
with code redemption.

``` text
Stolen authorization code
          +
No correct verifier
          ↓
        REJECT
```

PKCE therefore protects authorization-code redemption.

------------------------------------------------------------------------

## 10. Client Credentials

Used for machine-to-machine calls with no human user.

``` text
OrderService
     │ client authentication
     │ grant_type=client_credentials
     ▼
Authorization Server
     │
     ▼
Access Token
     │
     ▼
PaymentService
```

The token represents client/workload authority rather than a logged-in
user.

Client authentication can use mechanisms such as:

``` text
client_secret
private_key_jwt
mTLS
```

Refresh tokens are normally unnecessary for classic client credentials
because the service can authenticate again and obtain another access
token.

------------------------------------------------------------------------

## 11. OAuth Security Defenses

  Problem                               Defense
  ------------------------------------- -----------------------------------
  Authorization-code interception       PKCE
  Callback CSRF/transaction confusion   `state`
  ID Token replay/substitution          `nonce` + validation
  Redirect manipulation                 Strict redirect URI validation
  JWT modification                      Signature validation
  Wrong issuer                          `iss` validation
  Token for wrong API                   `aud` validation
  Expired token                         `exp` validation
  Excess authority                      Least-privilege scopes
  Bearer-token theft                    Short lifetime, DPoP/mTLS binding
  Refresh-token replay                  Rotation/reuse detection
  SPA secret leakage                    No static secret in public client

Mental model:

``` text
Authorization Request
       │
       ├── state ──► authorization response correlation
       ├── PKCE ───► code redemption
       └── nonce ──► OIDC ID Token/authentication result
```

### Bearer tokens

A bearer token means possession can be sufficient to use it.

Sender-constrained tokens strengthen this:

``` text
Token
  +
proof of private-key possession
        ↓
Access
```

Examples include DPoP and mTLS-bound access tokens.

------------------------------------------------------------------------

## 12. Spring Security Internals

Keep these three roles separate:

``` text
oauth2Login()
    → log user into Spring application

oauth2Client()
    → Spring obtains/uses tokens for outbound calls

oauth2ResourceServer()
    → Spring validates incoming access tokens
```

### `oauth2Login()`

``` text
Browser
   │ protected request
   ▼
SecurityFilterChain
   │
   ▼
OAuth2AuthorizationRequestRedirectFilter
   │
   ├─ build authorization request
   ├─ state
   ├─ nonce
   └─ PKCE where applicable
   ▼
Authorization Server
   │
   ▼
callback: code + state
   │
   ▼
OAuth2LoginAuthenticationFilter
   │
   ├─ correlate original request
   └─ exchange code
   ▼
Token Endpoint
   │
   ├─ Access Token
   └─ ID Token
   ▼
OIDC validation/user loading
   │
   ▼
OidcUser
   │
   ▼
OAuth2AuthenticationToken
   │
   ▼
SecurityContext
   │
   ▼
HTTP Session/JSESSIONID
```

### Resource Server

``` text
Client
   │ Bearer token
   ▼
BearerTokenAuthenticationFilter
   │
   ▼
AuthenticationManager
   │
   ├── JWT
   │    ↓
   │ JwtAuthenticationProvider
   │    ↓
   │ JwtDecoder
   │
   └── Opaque
        ↓
     OpaqueTokenAuthenticationProvider
        ↓
     OpaqueTokenIntrospector
```

------------------------------------------------------------------------

## 13. JWT Validation

JWT:

``` text
HEADER.PAYLOAD.SIGNATURE
```

Example header:

``` json
{
  "alg": "RS256",
  "kid": "KEY-A"
}
```

Example claims:

``` json
{
  "iss": "https://auth.example.com",
  "sub": "alice",
  "aud": "payment-api",
  "scope": "payments.read",
  "exp": 1786690000
}
```

### Signing

``` text
Authorization Server

Private Key
    │
    └── signs JWT


Resource Server

Public Key
    │
    └── verifies JWT
```

### JWKS

JWKS publishes public verification keys:

``` text
kid=A → Public Key A
kid=B → Public Key B
```

### Spring validation path

``` text
HTTP Request
     │ Bearer JWT
     ▼
BearerTokenAuthenticationFilter
     │
     ▼
JwtAuthenticationProvider
     │
     ▼
JwtDecoder
     │
     ├─ select trusted key using kid
     ├─ verify signature
     ├─ validate issuer
     ├─ validate timestamps
     ├─ validate configured audience
     └─ custom validators
     ▼
JwtAuthenticationToken
     │
     ▼
SecurityContext
```

### Is JWT validated on every request?

For a stateless Resource Server receiving a bearer JWT:

**Yes.**

Every authenticated bearer request presents the credential again, so
validation happens again.

But the Authorization Server/JWKS endpoint is **not normally called for
every request**.

``` text
Request 1 → JWT → cached public key → local validation
Request 2 → JWT → cached public key → local validation
Request 3 → JWT → cached public key → local validation
```

Remember:

``` text
JWT VALIDATION
    → every bearer request

JWKS NETWORK LOOKUP
    → only when required
```

### Key rotation

``` text
Initially:
kid=A → Public Key A

After rotation:
kid=A → Public Key A
kid=B → Public Key B
```

Old public keys can remain published during an overlap period so
still-valid older tokens continue to verify.

If a Resource Server encounters a new unknown `kid`, its JWK
infrastructure can refresh the key set.

### Issuer outage

If the required public key is already cached, the Resource Server may
continue locally validating tokens during a temporary issuer/JWKS
outage.

If a new unknown signing key must be fetched while JWKS is unavailable,
validation may fail.

------------------------------------------------------------------------

## 14. Opaque Token Validation

Opaque token:

``` text
3f8d9a76c45142c39e...
```

The Resource Server normally validates it through introspection.

``` text
Client
   │ opaque token
   ▼
Resource Server
   │
   │ introspection
   ▼
Authorization Server
   │
   └─ active? scopes? subject? expiry?
```

Spring:

``` text
BearerTokenAuthenticationFilter
        │
        ▼
OpaqueTokenAuthenticationProvider
        │
        ▼
OpaqueTokenIntrospector
        │
        ▼
Introspection Endpoint
```

Conceptual response:

``` json
{
  "active": true,
  "sub": "alice",
  "scope": "accounts.read",
  "exp": 1786690000
}
```

### Is it validated every request?

Normally, each bearer-token request can require introspection.

``` text
Request 1 → Resource Server → introspection
Request 2 → Resource Server → introspection
Request 3 → Resource Server → introspection
```

Caching can be added, but creates a trade-off:

``` text
More caching
   ↓
better performance
   +
slower revocation awareness
```

------------------------------------------------------------------------

## 15. JWT vs Opaque

  -----------------------------------------------------------------------
                          JWT                     Opaque
  ----------------------- ----------------------- -----------------------
  Claims embedded         Usually                 Not required

  Validation              Local cryptographic     Introspection

  Validate each bearer    Yes                     Yes
  request                                         

  Issuer call every       Normally no             Normally yes
  request                                         

  Key caching             Common                  Not the core model

  Immediate central       Harder                  Easier
  revocation                                      

  Runtime issuer          Lower                   Higher
  dependency                                      

  Microservice            Excellent               Requires scalable
  scalability                                     introspection
  -----------------------------------------------------------------------

Core picture:

``` text
JWT
──────────────────────────
Client
  │ JWT
  ▼
Resource Server
  │
  ├─ signature
  ├─ issuer
  ├─ audience
  ├─ expiry
  └─ scope
  ▼
LOCAL


OPAQUE
──────────────────────────
Client
  │ opaque token
  ▼
Resource Server
  │
  ▼
Authorization Server
  │ introspection
  ▼
active?
```

### Why JWT revocation is harder

``` text
10:00 token issued
10:15 user disabled
11:00 token expires
```

At 10:30 local validation can still show:

``` text
signature ✓
issuer    ✓
audience  ✓
expiry    ✓
```

The Resource Server does not inherently know that central account state
changed.

Typical mitigations include short-lived access tokens, refresh-token
revocation, sender-constrained tokens, selective central checks, or
special deny-list mechanisms.

------------------------------------------------------------------------

## 16. Microservice Token Propagation

Suppose:

``` text
User
 │ T1
 ▼
Service A
 │
 ▼
Service B
```

The correct first question is:

> **Whose authority is the downstream call exercising?**

### Token relay

``` text
User ── T1 ──► A ── T1 ──► B
```

Use only when T1 is actually intended for B.

If:

``` text
aud = service-a
```

B should reject it.

### Client Credentials

If B only needs A's service identity:

``` text
A
│ client_credentials
▼
Authorization Server
│
▼
service token
│
▼
B
```

### Token Exchange / On-Behalf-Of

If B needs delegated user context but a B-specific token:

``` text
User
 │ T1: aud=A
 ▼
A
 │ exchange
 ▼
Authorization Server
 │
 ▼
T2: aud=B
 │
 ▼
B
```

### Workload identity plus delegated identity

``` text
mTLS/service mesh
      ↓
Which service called me?

OAuth token
      ↓
Which user/delegated authority?
```

Decision:

  -----------------------------------------------------------------------
  Requirement                         Pattern
  ----------------------------------- -----------------------------------
  B needs A only                      Client Credentials

  B needs user and T1 is valid for B  Token relay

  B needs user with B-specific token  Token exchange/OBO

  B needs user + strong service       Delegated token + mTLS/workload
  identity                            identity
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 17. Logout and Session Lifecycle

These are independent:

``` text
Application Session
Access Token
Refresh Token
IdP SSO Session
```

Example:

``` text
App Session    |------30m------|
Access Token   |--15m--|
Refresh Token  |-------------8h-------------|
IdP Session    |----------------12h----------------|
```

### Local logout

``` text
POST /logout
      │
      ▼
LogoutFilter
      │
      ├─ clear SecurityContext
      ├─ invalidate HttpSession
      └─ remove local session
```

This does not necessarily terminate the IdP SSO session.

### OIDC provider logout

``` text
Spring App
   │
   ▼
OIDC Provider logout
   │
   └─ terminate SSO session
```

### Important distinction

``` text
Local logout
      ≠
JWT access-token revocation
      ≠
Refresh-token revocation
      ≠
IdP logout
```

A copied JWT may remain valid until expiry unless another
revocation/control mechanism exists.

------------------------------------------------------------------------

## 18. Scopes, Roles, Permissions and Claims

### Claim

Any asserted token information:

``` text
sub
aud
scope
department
tenant
```

### Scope

OAuth delegated API capability:

``` text
accounts.read
payments.create
```

### Role

Business/security classification:

``` text
CUSTOMER
TELLER
BRANCH_MANAGER
ADMIN
```

### Permission

Fine-grained capability:

``` text
PAYMENT_APPROVE
ACCOUNT_VIEW
```

### Spring `GrantedAuthority`

Spring's common authorization representation:

``` text
SCOPE_payments.read
ROLE_MANAGER
PAYMENT_APPROVE
```

`hasRole("ADMIN")` conventionally checks for:

``` text
ROLE_ADMIN
```

OAuth scopes commonly map as:

``` text
payments.read
       ↓
SCOPE_payments.read
```

and can be checked with:

``` java
@PreAuthorize("hasAuthority('SCOPE_payments.read')")
```

------------------------------------------------------------------------

## 19. RBAC, ABAC and Object Authorization

OAuth scopes usually provide coarse-grained access.

``` text
SCOPE_accounts.read
```

does not automatically mean:

``` text
may read every account
```

### RBAC

``` text
User
 ↓
Role
 ↓
Permissions
```

### ABAC

Uses attributes/context:

``` text
user.branch == payment.branch
AND
payment.amount <= user.approvalLimit
```

### Object-level authorization

For:

``` http
GET /accounts/12345
```

you may need:

``` text
JWT valid?
   ↓
audience correct?
   ↓
accounts.read scope?
   ↓
Does caller have access to account 12345?
   ↓
tenant/ownership/business policy?
   ↓
ALLOW
```

Spring can combine coarse and domain authorization:

``` java
@PreAuthorize(
    "hasAuthority('SCOPE_accounts.read') && " +
    "@accountAuthorization.canRead(authentication, #accountId)"
)
```

The important distinction is:

``` text
OAuth authorization
      ↓
May use this API capability

Business authorization
      ↓
May perform THIS operation
on THIS resource
```

------------------------------------------------------------------------

## 20. Architect Decision Guide

### Browser + BFF

``` text
Browser
   │ HttpOnly session cookie
   ▼
Spring BFF
   │ OAuth tokens server-side
   ▼
Backend APIs
```

Consider:

``` text
HttpOnly
Secure
SameSite
CSRF
CORS
session expiry
token refresh
OIDC logout
```

### Direct SPA

``` text
React/Angular
   │ Authorization Code + PKCE
   ▼
Authorization Server
   │
   ▼
Access Token
   │
   ▼
Resource API
```

A public SPA must not contain a static client secret.

### Service-to-service

``` text
Downstream needs only service?
       ↓
Client Credentials

Downstream needs user delegation?
       ↓
Token relay if appropriate
or Token Exchange/OBO

Need strong workload identity too?
       ↓
OAuth + mTLS/service identity
```

### Resource Server

Never stop at:

``` text
signature valid
```

Think:

``` text
signature
issuer
audience
timestamps
scope
business authorization
```

------------------------------------------------------------------------

## 21. Quick Revision

### OAuth vs OIDC

``` text
OAuth → API authorization
OIDC  → user authentication on OAuth
```

### Tokens

``` text
ID Token     → OIDC Client
Access Token → Resource Server
Refresh Token→ Authorization Server/token endpoint
```

### `state`, PKCE, `nonce`

``` text
state → authorization transaction correlation / CSRF defense

PKCE → authorization-code redemption protection

nonce → bind OIDC authentication result/ID Token
```

### JWT

``` text
BearerTokenAuthenticationFilter
        ↓
JwtAuthenticationProvider
        ↓
JwtDecoder
        ↓
signature + claim validation
        ↓
JwtAuthenticationToken
        ↓
SecurityContext
```

Validated on every bearer-token request, normally using cached public
keys.

### Opaque

``` text
BearerTokenAuthenticationFilter
        ↓
OpaqueTokenAuthenticationProvider
        ↓
OpaqueTokenIntrospector
        ↓
Authorization Server
```

Normally introspected for each bearer-token request unless an explicit
cache changes that behavior.

### One-sentence JWT vs opaque

> JWT favors decentralized/local validation and scalability; opaque
> tokens favor centralized token-state control at the cost of
> introspection dependency.

### Scope vs role

``` text
Scope
  → OAuth delegated capability

Role
  → business/security classification

Permission
  → fine-grained capability

GrantedAuthority
  → Spring authorization representation
```

### Final mental model

``` text
Client
  │
  │ TLS / mTLS
  ▼
Gateway / Ingress
  │
  ▼
Resource Server
  │
  ├── Authenticate credential
  │      ├─ JWT → local validation
  │      └─ Opaque → introspection
  │
  ├── Validate token policy
  │      ├─ issuer
  │      ├─ audience
  │      ├─ expiry
  │      └─ signature/active state
  │
  ├── OAuth authorization
  │      └─ scopes
  │
  └── Business authorization
         ├─ roles
         ├─ permissions
         ├─ ownership
         ├─ tenant
         └─ ABAC/business rules
```

## Core Principles

1.  Authentication and authorization solve different problems.
2.  OAuth is primarily authorization; OIDC adds authentication.
3.  Access Tokens go to Resource Servers; ID Tokens go to OIDC clients.
4.  PKCE, `state` and `nonce` solve different security problems.
5.  JWT signature validation alone is insufficient; validate issuer,
    audience, lifetime and policy.
6.  JWTs are normally validated locally on every bearer-token request.
7.  JWKS is not normally contacted on every request because verification
    keys are cached.
8.  Opaque tokens normally use introspection and therefore have greater
    runtime dependency on the Authorization Server.
9.  Never blindly relay one JWT through every microservice.
10. Choose downstream credentials based on whose authority the call
    exercises.
11. OAuth scopes do not replace object/domain authorization.
12. Logout, token expiry, token revocation and IdP-session termination
    are separate.
13. CORS is a browser security policy, not authentication.
14. mTLS can establish workload identity while OAuth independently
    establishes API authorization.
15. Apply least privilege at every security boundary.
