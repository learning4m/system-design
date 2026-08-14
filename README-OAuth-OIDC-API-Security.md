# OAuth 2.0, OpenID Connect, JWT, mTLS & API Security --- Detailed Learning Handbook

> **Purpose:** This is a study handbook, not a summary. It starts from
> basic API-security mechanisms and builds toward OAuth 2.0, OIDC,
> JWT/opaque token validation, Spring Security internals, microservice
> delegation, and enterprise authorization. Each major concept answers
> four questions: **What is it? Why does it exist? How does it work?
> What can go wrong?**

------------------------------------------------------------------------

# Table of Contents

1.  [The Security Layers](#1-the-security-layers)
2.  [Authentication vs
    Authorization](#2-authentication-vs-authorization)
3.  [TLS and HTTPS](#3-tls-and-https)
4.  [Mutual TLS --- mTLS](#4-mutual-tls--mtls)
5.  [mTLS and the Servlet X.509
    Certificate](#5-mtls-and-the-servlet-x509-certificate)
6.  [mTLS When TLS Terminates at
    Ingress](#6-mtls-when-tls-terminates-at-ingress)
7.  [API Keys](#7-api-keys)
8.  [HTTP Basic Authentication](#8-http-basic-authentication)
9.  [Request Signing](#9-request-signing)
10. [Session and Cookie
    Authentication](#10-session-and-cookie-authentication)
11. [Cookie Security Attributes](#11-cookie-security-attributes)
12. [Same Origin and CORS](#12-same-origin-and-cors)
13. [`Access-Control-Allow-Credentials`](#13-access-control-allow-credentials)
14. [CORS Preflight](#14-cors-preflight)
15. [Why OAuth 2.0 Exists](#15-why-oauth-20-exists)
16. [OAuth 2.0 Actors](#16-oauth-20-actors)
17. [OAuth Authorization vs User
    Authentication](#17-oauth-authorization-vs-user-authentication)
18. [OpenID Connect](#18-openid-connect)
19. [Access Token vs ID Token vs Refresh
    Token](#19-access-token-vs-id-token-vs-refresh-token)
20. [Authorization Code Flow](#20-authorization-code-flow)
21. [PKCE in Depth](#21-pkce-in-depth)
22. [`state`, `nonce` and PKCE](#22-state-nonce-and-pkce)
23. [Client Types: Public vs
    Confidential](#23-client-types-public-vs-confidential)
24. [Client Credentials Flow](#24-client-credentials-flow)
25. [Resource Server](#25-resource-server)
26. [Bearer Tokens](#26-bearer-tokens)
27. [JWT Structure](#27-jwt-structure)
28. [JWT Signing and Verification](#28-jwt-signing-and-verification)
29. [JWT Claims and Their
    Validation](#29-jwt-claims-and-their-validation)
30. [JWKS and `kid`](#30-jwks-and-kid)
31. [Key Rotation](#31-key-rotation)
32. [JWT Validation on Every
    Request](#32-jwt-validation-on-every-request)
33. [Opaque Tokens](#33-opaque-tokens)
34. [Token Introspection](#34-token-introspection)
35. [Opaque Validation on Every
    Request](#35-opaque-validation-on-every-request)
36. [JWT vs Opaque Tokens](#36-jwt-vs-opaque-tokens)
37. [JWT Revocation Problem](#37-jwt-revocation-problem)
38. [Refresh Tokens and Rotation](#38-refresh-tokens-and-rotation)
39. [Spring Security OAuth Roles](#39-spring-security-oauth-roles)
40. [Spring Resource Server --- JWT
    Internals](#40-spring-resource-server--jwt-internals)
41. [Spring Resource Server --- Opaque
    Internals](#41-spring-resource-server--opaque-internals)
42. [Spring `oauth2Login()` Internals](#42-spring-oauth2login-internals)
43. [Scopes, Roles, Permissions and
    Claims](#43-scopes-roles-permissions-and-claims)
44. [Spring `GrantedAuthority`](#44-spring-grantedauthority)
45. [RBAC](#45-rbac)
46. [ABAC](#46-abac)
47. [Object-Level Authorization](#47-object-level-authorization)
48. [Gateway vs Service
    Authorization](#48-gateway-vs-service-authorization)
49. [Token Propagation Across
    Microservices](#49-token-propagation-across-microservices)
50. [Token Relay](#50-token-relay)
51. [Client Credentials for Downstream
    Calls](#51-client-credentials-for-downstream-calls)
52. [Token Exchange / On-Behalf-Of](#52-token-exchange--on-behalf-of)
53. [mTLS + OAuth Together](#53-mtls--oauth-together)
54. [Logout and the Four Independent
    Lifecycles](#54-logout-and-the-four-independent-lifecycles)
55. [Complete Browser-to-Microservice
    Example](#55-complete-browser-to-microservice-example)
56. [Common Misconceptions](#56-common-misconceptions)
57. [Architect-Level Decision
    Framework](#57-architect-level-decision-framework)
58. [Spring Configuration Examples](#58-spring-configuration-examples)
59. [Interview Questions and
    Answers](#59-interview-questions-and-answers)
60. [Final Memory Map](#60-final-memory-map)

------------------------------------------------------------------------

# 1. The Security Layers

Suppose an enterprise application looks like this:

``` text
Browser / React
      │
      ▼
Kubernetes Ingress
      │
      ▼
Spring Cloud Gateway
      │
      ▼
Order Service
      │
      ▼
Payment Service
      │
      ▼
Database
```

"Secure the application" is not one requirement. We need to answer
several independent questions.

### Question 1 --- Can an attacker read network traffic?

Solved primarily by:

``` text
TLS / HTTPS
```

### Question 2 --- Which service established the connection?

Possible mechanism:

``` text
mTLS
```

### Question 3 --- Who is the human user?

Possible mechanisms:

``` text
Username/password + session
OIDC
Certificate authentication
etc.
```

### Question 4 --- Is a client allowed to call an API?

Possible mechanisms:

``` text
API key
OAuth Access Token
mTLS identity + policy
```

### Question 5 --- What API operations may it perform?

``` text
OAuth scopes
permissions
roles
```

### Question 6 --- Can Alice access THIS account?

``` text
object-level/domain authorization
RBAC + ABAC + ownership
```

This gives the first important mental model:

``` text
NETWORK SECURITY
       ↓
IDENTITY / AUTHENTICATION
       ↓
TOKEN/CREDENTIAL VALIDATION
       ↓
COARSE-GRAINED API AUTHORIZATION
       ↓
FINE-GRAINED BUSINESS AUTHORIZATION
```

------------------------------------------------------------------------

# 2. Authentication vs Authorization

These two words must remain separate.

## Authentication

> **Who are you?**

Examples:

``` text
Alice authenticated using OIDC
Order Service authenticated using mTLS
Batch Job authenticated using client credentials
```

## Authorization

> **What are you allowed to do?**

Examples:

``` text
Alice may read accounts.
Alice may not approve a ₹1 crore payment.
Order Service may call Payment API.
```

A request can be successfully authenticated and still be denied:

``` text
Authentication
     ✓
Alice is Alice
     │
     ▼
Authorization
     ✗
Alice cannot access Bob's account
```

------------------------------------------------------------------------

# 3. TLS and HTTPS

TLS operates below HTTP application authentication.

Simplified connection:

``` text
Browser                                  Server
   │                                       │
   │──── ClientHello ─────────────────────►│
   │                                       │
   │◄──── certificate + handshake ─────────│
   │                                       │
   │ verify server certificate             │
   │                                       │
   │════ encrypted connection established ═│
   │                                       │
   │──── GET /accounts HTTP/1.1 ──────────►│
```

Notice that HTTP comes **after** the secure TLS channel is established.

TLS provides three important properties:

### Confidentiality

Someone observing the network should not be able to read the plaintext.

### Integrity

An intermediary should not be able to silently modify the traffic.

### Server authentication

The client validates that it connected to the intended server identity.

TLS by itself does not mean:

``` text
Alice is authorized to transfer money.
```

It only protects/authenticates the communication channel according to
its configured certificate model.

------------------------------------------------------------------------

# 4. Mutual TLS --- mTLS

Normal HTTPS normally authenticates the server to the client.

mTLS additionally requires the client to present a certificate.

``` text
Client                                  Server
  │                                       │
  │◄──── server certificate ───────────────│
  │                                       │
  │──── client certificate ───────────────►│
  │                                       │
  │     certificate/private-key proof     │
  │                                       │
  │════ encrypted authenticated channel ══│
```

Suppose Order Service has:

``` text
Certificate:
Subject CN = order-service
SAN = spiffe/order-service
Issuer = Enterprise Internal CA
```

Payment Service trusts the enterprise CA.

The TLS layer validates that:

1.  the client certificate chains to a trusted CA;
2.  the certificate is valid for the relevant policy/time;
3.  the client proves possession of the matching private key.

Only after this should the application treat certificate identity as
trustworthy.

------------------------------------------------------------------------

# 5. mTLS and the Servlet X.509 Certificate

If TLS terminates at the Java web server:

``` text
Client
   │ mTLS
   ▼
Tomcat
   │
   ▼
Spring MVC
```

Tomcat participated in the handshake.

The servlet request can expose the authenticated certificate chain:

``` java
X509Certificate[] certificates =
    (X509Certificate[]) request.getAttribute(
        "jakarta.servlet.request.X509Certificate"
    );
```

You might then inspect:

``` java
X509Certificate client = certificates[0];

String subject =
    client.getSubjectX500Principal().getName();
```

Example:

``` text
CN=order-service,OU=Payments,O=MyCompany
```

But the logic is not:

``` text
if CN == "order-service"
    trust request
```

The real chain is:

``` text
Certificate cryptographically validated
        ↓
Certificate identity extracted
        ↓
Identity mapped to application principal
        ↓
Authorization policy evaluated
```

------------------------------------------------------------------------

# 6. mTLS When TLS Terminates at Ingress

This is an important production distinction.

``` text
Client
  │
  │ mTLS #1
  ▼
Ingress
  │
  │ HTTP or TLS #2
  ▼
Spring Boot
```

TLS #1 terminates at Ingress.

Spring Boot is not a participant in TLS #1.

Therefore:

``` java
request.getAttribute("jakarta.servlet.request.X509Certificate")
```

cannot automatically represent the original external TLS peer.

## Pattern A --- Ingress validates certificate and forwards identity

``` text
Client
   │ certificate
   ▼
Ingress
   │
   ├─ validate certificate
   ├─ validate CA
   ├─ inspect SAN/Subject
   └─ create trusted identity metadata
          │
          ▼
Spring Boot
```

For example:

``` http
X-Verified-Client-Subject: CN=order-service
```

The trust boundary must guarantee:

``` text
Internet caller
     │
     ├── cannot directly access backend
     │
     └── cannot supply trusted header unchanged

Ingress
     │
     └── strips and recreates trusted header
```

Otherwise:

``` http
X-Verified-Client-Subject: CN=admin
```

would be trivially spoofable.

## Pattern B --- TLS passthrough

``` text
Client
  │ encrypted mTLS
  ▼
Ingress
  │ pass encrypted bytes
  ▼
Tomcat
  │ terminates mTLS
```

Now Java sees the real TLS client certificate.

------------------------------------------------------------------------

# 7. API Keys

An API key is typically a random credential issued to a client
application.

``` http
GET /weather
X-API-Key: KJH872...
```

Server:

``` text
Receive key
   ↓
Lookup key
   ↓
Is active?
   ↓
Which client?
   ↓
Which quota/policy?
```

Useful for:

``` text
simple integrations
developer APIs
application identification
rate-limit plans
```

API keys usually do not provide the rich delegation model of OAuth.

### API key vs password

An API key often identifies an application rather than a human user.

### API key vs OAuth client secret

API key:

``` text
Client ── API Key ──► Business API
```

OAuth client secret:

``` text
Client
  │ client secret
  ▼
Authorization Server
  │
  ▼
Access Token
  │
  ▼
Business API
```

Do not send the OAuth client secret to every business API.

------------------------------------------------------------------------

# 8. HTTP Basic Authentication

Basic authentication sends:

``` http
Authorization: Basic base64(username:password)
```

Base64 is **encoding, not encryption**.

Therefore Basic authentication must be protected by TLS.

Conceptually:

``` text
username:password
      ↓
Base64
      ↓
Authorization header
      ↓
HTTPS protects transport
```

Unlike a server-side session, credentials are typically supplied
repeatedly.

------------------------------------------------------------------------

# 9. Request Signing

Request signing protects more than "I possess a token."

Suppose client and server share secret `K`.

Client canonicalizes:

``` text
POST
/payments
SHA256(body)
timestamp
nonce
```

Then:

``` text
signature = HMAC(K, canonical-request)
```

Request:

``` http
POST /payments
X-Timestamp: ...
X-Nonce: ...
X-Signature: ...
```

Server performs the same HMAC calculation.

``` text
received signature
       │
       ▼
recompute using shared K
       │
       ▼
constant-time compare
```

Yes, with HMAC the same secret must be securely provisioned to both
parties first.

## Replay defense

If an attacker captures the exact request, a signature alone may remain
valid.

So servers commonly check:

``` text
timestamp within allowed window?
nonce/request ID already used?
```

Asymmetric signing instead uses:

``` text
Client private key → sign
Server public key → verify
```

------------------------------------------------------------------------

# 10. Session and Cookie Authentication

A traditional Spring application:

``` text
Browser
   │ POST /login
   ▼
Spring Security
   │
   ├─ authenticate credentials
   ├─ create Authentication
   ├─ store SecurityContext in session
   └─ return JSESSIONID
```

Response:

``` http
Set-Cookie: JSESSIONID=ABC123
```

Next request:

``` http
GET /profile
Cookie: JSESSIONID=ABC123
```

Server:

``` text
ABC123
  ↓
HttpSession
  ↓
SecurityContext
  ↓
Authentication
```

The browser can send cookies whether the frontend was built with:

``` text
React
Angular
Vue
plain JavaScript
```

Framework choice does not remove browser cookie functionality.

------------------------------------------------------------------------

# 11. Cookie Security Attributes

Example:

``` http
Set-Cookie:
JSESSIONID=ABC;
HttpOnly;
Secure;
SameSite=Lax
```

## HttpOnly

Prevents normal JavaScript APIs from reading the cookie.

``` text
document.cookie
      │
      └── HttpOnly cookie hidden
```

But:

``` text
Browser HTTP request
      │
      └── cookie may still be attached
```

That is the entire point: JavaScript cannot easily steal it, while the
browser can still use it for authentication.

## Secure

Cookie is sent only over secure transport.

## SameSite

Controls cross-site cookie behavior.

Simplified:

``` text
Strict → very restrictive
Lax    → allows limited cross-site navigation cases
None   → allows cross-site use; normally requires Secure
```

SameSite is especially relevant to CSRF defense.

------------------------------------------------------------------------

# 12. Same Origin and CORS

An origin consists of:

``` text
scheme + host + port
```

Examples:

``` text
https://app.example.com
https://api.example.com
```

Different host → different origin.

Browsers enforce the Same-Origin Policy around script access to
cross-origin resources.

CORS lets a server selectively relax that restriction.

Server might return:

``` http
Access-Control-Allow-Origin: https://app.example.com
```

Important:

> CORS is a browser security policy. It is not API authentication.

A non-browser client such as a backend service does not rely on browser
CORS enforcement.

------------------------------------------------------------------------

# 13. `Access-Control-Allow-Credentials`

Frontend:

``` javascript
fetch("https://api.example.com/me", {
    credentials: "include"
});
```

This tells the browser:

> Include applicable credentials, such as cookies, for this cross-origin
> request when permitted.

Server:

``` http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

The server response tells the browser that the response may be exposed
to credentialed cross-origin JavaScript if the rest of the CORS policy
passes.

## What if the server does NOT send it?

Do not think:

``` text
No ACA-Credentials
      ↓
request never reaches controller
```

The actual flow can be:

``` text
Browser sends request
       ↓
Server receives request
       ↓
Server may process/change data
       ↓
Server sends response
       ↓
Browser checks CORS response headers
       ↓
CORS fails
       ↓
JavaScript cannot read response
```

This is why CORS cannot replace CSRF protection or server-side
authorization.

------------------------------------------------------------------------

# 14. CORS Preflight

The browser determines whether a cross-origin request qualifies as a
CORS "simple request."

If it does not, browser sends an `OPTIONS` preflight.

Example intended request:

``` http
POST /payments
Authorization: Bearer ...
Content-Type: application/json
```

Browser may first send:

``` http
OPTIONS /payments
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization,content-type
```

Server responds with allowed origin/method/headers.

Only then does the browser decide whether to send the actual request.

The JavaScript application does not manually decide "this needs
preflight." The browser's CORS implementation does.

------------------------------------------------------------------------

# 15. Why OAuth 2.0 Exists

Suppose Alice uses a budgeting application that needs bank transactions.

Without delegation:

``` text
Alice
  │ gives bank password
  ▼
Budget App
  │ impersonates Alice
  ▼
Bank
```

This is dangerous.

The Budget App gets too much power and knows Alice's password.

OAuth instead allows:

``` text
Alice
  │
  │ authorize "read transactions"
  ▼
Bank Authorization Server
  │
  │ issue limited Access Token
  ▼
Budget App
  │
  │ Access Token
  ▼
Bank Resource API
```

The Access Token can be limited by:

``` text
scope
audience/resource
lifetime
client
subject
other policy
```

OAuth's central idea is therefore:

> Give a client limited delegated access without giving it the resource
> owner's password.

------------------------------------------------------------------------

# 16. OAuth 2.0 Actors

Four fundamental roles:

## Resource Owner

Usually the user whose data/access is involved.

``` text
Alice
```

## Client

Application requesting access.

``` text
Budget App
```

## Authorization Server

Authenticates/authorizes and issues tokens.

``` text
login.bank.com
```

## Resource Server

Protected API accepting access tokens.

``` text
api.bank.com
```

Complete picture:

``` text
Alice
(Resource Owner)
      │
      ▼
Budget App
(Client)
      │
      │ authorization
      ▼
Authorization Server
      │
      │ Access Token
      ▼
Budget App
      │
      │ Access Token
      ▼
Bank API
(Resource Server)
```

------------------------------------------------------------------------

# 17. OAuth Authorization vs User Authentication

OAuth does not fundamentally mean:

``` text
"This is Alice."
```

It fundamentally addresses delegated access.

An Access Token may represent:

``` text
user delegation
client/service authority
another authorized subject/context
```

That is why using OAuth Access Tokens as a generic "login token" without
an identity protocol creates ambiguity.

OIDC was created to add standardized authentication semantics.

------------------------------------------------------------------------

# 18. OpenID Connect

OIDC builds an authentication layer using OAuth mechanisms.

OAuth:

``` text
What access is granted?
```

OIDC:

``` text
Who authenticated?
```

OIDC introduces the **ID Token** and standardized identity-related
behavior.

Example ID Token claims:

``` json
{
  "iss": "https://login.example.com",
  "sub": "248289761001",
  "aud": "my-web-client",
  "exp": 1786690000,
  "iat": 1786686400,
  "nonce": "..."
}
```

The client validates the ID Token and establishes the user's
authenticated identity.

------------------------------------------------------------------------

# 19. Access Token vs ID Token vs Refresh Token

This is critical.

## Access Token

Recipient:

``` text
Resource Server
```

Used like:

``` http
Authorization: Bearer <access-token>
```

Question it helps answer:

``` text
What API access is granted?
```

## ID Token

Recipient:

``` text
OIDC Client
```

Question:

``` text
Who authenticated, and what authentication assertion did the issuer make?
```

The Resource Server should not use an ID Token merely because it happens
to be a JWT.

## Refresh Token

Recipient:

``` text
Authorization Server token endpoint
```

Purpose:

``` text
obtain new Access Token
```

Diagram:

``` text
Authorization Server
   │
   ├──── ID Token ─────► Client
   │
   ├── Access Token ───► Client ───► Resource Server
   │
   └── Refresh Token ──► Client ───► Token Endpoint later
```

------------------------------------------------------------------------

# 20. Authorization Code Flow

Detailed flow:

``` text
User Browser             Client                Authorization Server
    │                      │                           │
    │ GET protected page   │                           │
    │─────────────────────►│                           │
    │                      │                           │
    │ redirect to /authorize                           │
    │◄─────────────────────│                           │
    │                                                  │
    │ authorization request                            │
    │─────────────────────────────────────────────────►│
    │                                                  │
    │                  authenticate user               │
    │                  consent/policy                  │
    │                                                  │
    │ redirect with code                               │
    │◄─────────────────────────────────────────────────│
    │                                                  │
    │ callback?code=ABC                                │
    │─────────────────────►│                           │
    │                      │                           │
    │                      │ POST token endpoint       │
    │                      │ code=ABC                  │
    │                      │──────────────────────────►│
    │                      │                           │
    │                      │ access/id/refresh tokens  │
    │                      │◄──────────────────────────│
```

The authorization code is an intermediate credential.

The client exchanges it for tokens through the token endpoint.

------------------------------------------------------------------------

# 21. PKCE in Depth

PKCE addresses authorization-code interception.

Client creates:

``` text
code_verifier = cryptographically random secret
```

Then:

``` text
code_challenge =
BASE64URL(SHA256(code_verifier))
```

Authorization request:

``` text
code_challenge = C
code_challenge_method = S256
```

Authorization Server binds C to the authorization transaction/code.

Later token request:

``` text
code = ABC
code_verifier = V
```

Server computes:

``` text
BASE64URL(SHA256(V))
```

and compares it to C.

Attack:

``` text
Attacker steals code ABC
       │
       ▼
Attacker calls token endpoint
       │
       ├─ has ABC
       └─ does not have V
       │
       ▼
Server cannot match challenge
       │
       ▼
REJECT
```

PKCE does not replace TLS.

PKCE does not authenticate the user.

PKCE does not replace `state`.

It specifically protects the **code redemption** step.

------------------------------------------------------------------------

# 22. `state`, `nonce` and PKCE

These are often confused.

## `state`

Client generates random state before redirect.

``` text
state=S123
```

Authorization response must return the matching value.

Purpose:

``` text
correlate request/response
help defend against authorization-response CSRF/confusion
```

## PKCE

``` text
challenge in authorization request
verifier in token request
```

Purpose:

``` text
bind authorization code redemption
to the initiating client instance
```

## `nonce`

OIDC client sends nonce in authentication request.

ID Token contains corresponding nonce.

Purpose:

``` text
bind ID Token/authentication result
to the initiating authentication request
and mitigate replay/substitution scenarios
```

Memory:

``` text
state → callback transaction

PKCE  → authorization code

nonce → ID Token / OIDC authentication
```

------------------------------------------------------------------------

# 23. Client Types: Public vs Confidential

## Confidential client

Can keep credentials confidential.

Examples:

``` text
server-side Spring application
backend service
```

Can potentially authenticate using:

``` text
client_secret
private_key_jwt
mTLS
```

## Public client

Cannot safely keep a static secret.

Examples:

``` text
SPA JavaScript bundle
native mobile app
```

Why can't a SPA keep a secret?

Because anything shipped to the browser can ultimately be inspected by
the user/attacker.

Therefore:

``` text
client_secret embedded in React bundle
```

is not truly secret.

PKCE is essential for public-client authorization-code flows.

------------------------------------------------------------------------

# 24. Client Credentials Flow

No human user:

``` text
Order Service                        Authorization Server
     │                                      │
     │ POST /token                          │
     │ grant_type=client_credentials        │
     │ client authentication                │
     │─────────────────────────────────────►│
     │                                      │
     │ Access Token                         │
     │◄─────────────────────────────────────│
```

Then:

``` text
Order Service
     │ Authorization: Bearer <service-token>
     ▼
Payment Service
```

The resulting authority belongs to the client/service context.

This is different from:

``` text
Alice delegated access to Order Service
```

There is no Alice in the basic client-credentials flow.

------------------------------------------------------------------------

# 25. Resource Server

A Resource Server is the protected API.

Suppose:

``` text
Authorization Server = login.example.com
Resource Server      = payments.example.com
```

The Resource Server receives:

``` http
POST /payments
Authorization: Bearer <access-token>
```

Its responsibilities include:

``` text
Validate/authenticate token
        ↓
Construct caller Authentication
        ↓
Enforce scope/authority
        ↓
Enforce business authorization
        ↓
Run controller/service logic
```

It should not simply decode the token and trust whatever fields appear.

------------------------------------------------------------------------

# 26. Bearer Tokens

A bearer token has a fundamental property:

> Whoever possesses a valid bearer token may be able to use it.

It is similar conceptually to:

``` text
"Bearer of this credential gets the represented authority."
```

Therefore token theft matters.

Defenses include:

``` text
TLS
short lifetimes
safe storage
avoid logging tokens
refresh-token controls
sender-constrained tokens
```

mTLS-bound access tokens and DPoP can constrain token use to a party
proving possession of an associated private key.

------------------------------------------------------------------------

# 27. JWT Structure

JWT commonly looks like:

``` text
xxxxx.yyyyy.zzzzz
```

Three parts:

``` text
Header
Payload
Signature
```

Header:

``` json
{
  "alg": "RS256",
  "kid": "key-2026-08"
}
```

Payload:

``` json
{
  "iss": "https://auth.example.com",
  "sub": "alice",
  "aud": "payment-api",
  "scope": "payments.read payments.create",
  "iat": 1786686400,
  "exp": 1786690000
}
```

Important:

> JWT payload is commonly Base64URL encoded, not encrypted.

Therefore do not put secrets into claims merely because the token "looks
scrambled."

------------------------------------------------------------------------

# 28. JWT Signing and Verification

With asymmetric signing:

``` text
Authorization Server
       │
       │ private key
       ▼
sign JWT
```

Resource Server:

``` text
JWT
 │
 ▼
public key
 │
 ▼
verify signature
```

Why public/private keys?

The Authorization Server must be able to create valid signatures.

Resource Servers only need to verify them.

Therefore distributing public keys does not give Resource Servers the
power to mint valid issuer tokens.

### Tampering example

Original:

``` json
"scope": "payments.read"
```

Attacker edits:

``` json
"scope": "payments.admin"
```

The signed bytes changed.

The old signature no longer matches.

Resource Server rejects the token.

------------------------------------------------------------------------

# 29. JWT Claims and Their Validation

## `iss` --- issuer

``` json
"iss": "https://auth.example.com"
```

Question:

``` text
Which issuer claims to have created this token?
```

Resource Server must trust/configure the expected issuer.

## `sub` --- subject

``` json
"sub": "alice-id-123"
```

Identifies the subject in the issuer's namespace.

## `aud` --- audience

``` json
"aud": "payment-api"
```

Question:

``` text
Which resource/API is this token intended for?
```

A valid signature is not enough if:

``` text
aud = order-api
```

but the token is presented to:

``` text
payment-api
```

## `exp`

Expiry.

``` text
current time > exp
       ↓
reject
```

## `nbf`

Not-before.

Token should not be accepted before the permitted time.

## `iat`

Issued-at time.

Useful context but not by itself the complete acceptance policy.

## Scope claim

May be represented as:

``` json
"scope": "payments.read payments.create"
```

or issuer-specific alternatives.

Resource Server maps trusted scope information into authorization
decisions.

------------------------------------------------------------------------

# 30. JWKS and `kid`

JWKS = JSON Web Key Set.

It exposes public verification keys.

Conceptually:

``` text
JWKS
 ├─ kid=A → Public Key A
 └─ kid=B → Public Key B
```

JWT:

``` json
{
  "alg": "RS256",
  "kid": "B"
}
```

Resource Server:

``` text
read kid B
   ↓
find Public Key B
   ↓
verify signature
```

The `kid` is a key identifier; it is not itself the cryptographic key.

------------------------------------------------------------------------

# 31. Key Rotation

Why rotate signing keys?

``` text
security hygiene
planned lifecycle
response to compromise
cryptographic policy
```

Suppose current key is A.

``` text
Private A signs tokens
Public A published
```

Issuer generates B.

During transition:

``` text
JWKS:
A → Public A
B → Public B
```

New tokens:

``` text
kid=B
```

Old tokens:

``` text
kid=A
```

still validate while A remains published and those tokens remain valid.

After the maximum relevant old-token lifetime passes, A can be retired.

------------------------------------------------------------------------

# 32. JWT Validation on Every Request

This was the central question.

Suppose the same JWT is sent three times:

``` text
Request 1 → JWT-X
Request 2 → JWT-X
Request 3 → JWT-X
```

For a stateless Resource Server:

``` text
Request 1 → validate JWT-X
Request 2 → validate JWT-X
Request 3 → validate JWT-X
```

**Yes, validation occurs again for each bearer-token request.**

But:

``` text
validate token
```

does not equal:

``` text
contact Authorization Server
```

Normal JWT path:

``` text
JWT
 │ kid=A
 ▼
Resource Server
 │
 ├─ Public Key A already cached
 │
 ├─ verify signature locally
 │
 ├─ validate exp
 │
 ├─ validate iss
 │
 ├─ validate configured aud
 │
 └─ build Authentication
```

### What is cached?

Normally:

``` text
Public Key / JWK Set
```

not:

``` text
"JWT-X was valid yesterday so skip validation today."
```

That distinction is essential.

------------------------------------------------------------------------

# 33. Opaque Tokens

Opaque token:

``` text
sH38fj20dK...
```

There is no requirement that the Resource Server can interpret its
contents.

Think of it as:

``` text
reference / handle
```

The actual authorization state can remain at the Authorization Server.

------------------------------------------------------------------------

# 34. Token Introspection

Resource Server receives:

``` http
Authorization: Bearer OPAQUE-X
```

Then:

``` text
Resource Server
      │
      │ token=OPAQUE-X
      ▼
Authorization Server
/introspect
      │
      ├─ token found?
      ├─ active?
      ├─ expired?
      ├─ revoked?
      ├─ subject?
      └─ scopes?
```

Response may look like:

``` json
{
  "active": true,
  "sub": "alice",
  "scope": "payments.read",
  "exp": 1786690000
}
```

The Resource Server uses the response to establish
authentication/authorities.

The Resource Server itself normally authenticates to the introspection
endpoint according to the Authorization Server's policy.

------------------------------------------------------------------------

# 35. Opaque Validation on Every Request

Normal direct introspection:

``` text
Request 1
  ↓
Resource Server
  ↓
Introspection


Request 2
  ↓
Resource Server
  ↓
Introspection
```

So yes, opaque validation can create a network call per request.

An implementation can cache:

``` text
OPAQUE-X → active=true, scope=...
```

But suppose:

``` text
10:00 cached active=true for 5 min
10:01 token revoked centrally
```

Resource Server may still trust its cache until expiration.

Therefore:

``` text
more caching
    ↓
lower latency/load
    +
slower revocation visibility
```

------------------------------------------------------------------------

# 36. JWT vs Opaque Tokens

## JWT architecture

``` text
                    Authorization Server
                          │
                    Public Keys/JWKS
                          │
                          ▼
                      cached keys

Client ── JWT ──► Resource Server
                     │
                     └── validate locally
```

## Opaque architecture

``` text
Client ── opaque ──► Resource Server
                         │
                         ▼
                 Authorization Server
                    introspection
```

  -----------------------------------------------------------------------
  Property                JWT                     Opaque
  ----------------------- ----------------------- -----------------------
  Self-contained claims   Usually                 No requirement

  Per-request validation  Yes                     Yes

  Per-request issuer call Normally no             Normally yes

  Revocation visibility   Harder                  Easier

  Runtime coupling        Lower                   Higher

  Latency                 Lower validation        Includes introspection
                          overhead                

  Token information       Claims usually          Minimal
  exposed to holder       decodable               

  Scaling model           Decentralized           Central introspection
                                                  must scale
  -----------------------------------------------------------------------

Neither is universally "better."

Architecture decides based on requirements.

------------------------------------------------------------------------

# 37. JWT Revocation Problem

JWT's strength creates its revocation challenge.

Example:

``` text
10:00 JWT issued
exp = 11:00

10:15 Alice account disabled

10:30 API receives JWT
```

Resource Server locally checks:

``` text
signature ✓
iss       ✓
aud       ✓
exp       ✓
```

How does it know Alice was disabled at 10:15?

It does not automatically know.

Possible strategies include:

``` text
short-lived Access Tokens
central checks for high-risk actions
deny/block lists for exceptional cases
sender-constrained tokens
revoke refresh token/session to prevent renewal
```

This is a consistency trade-off of decentralized validation.

------------------------------------------------------------------------

# 38. Refresh Tokens and Rotation

Access Tokens should often be relatively short-lived.

Instead of forcing the user to authenticate every 10 minutes:

``` text
Access Token expires
       ↓
Client presents Refresh Token
       ↓
Authorization Server
       ↓
New Access Token
```

Refresh Tokens are more sensitive because they can create future Access
Tokens.

## Rotation

Instead of repeatedly using:

``` text
Refresh Token R1 forever
```

the server can do:

``` text
R1 used
 ↓
R1 invalidated
 ↓
R2 issued
```

Next:

``` text
R2 used
 ↓
R2 invalidated
 ↓
R3 issued
```

If old R1 appears again, that can indicate replay/theft and trigger
protective action according to provider policy.

------------------------------------------------------------------------

# 39. Spring Security OAuth Roles

Three configurations often get mixed up.

## `oauth2Login()`

``` text
Spring app acts as OAuth/OIDC Client
for logging a human user into the application.
```

## `oauth2Client()`

``` text
Spring app acts as OAuth Client
for outbound API calls.
```

## `oauth2ResourceServer()`

``` text
Spring app acts as protected API
and validates incoming Access Tokens.
```

One application can use more than one, but they solve different
problems.

------------------------------------------------------------------------

# 40. Spring Resource Server --- JWT Internals

Incoming request:

``` http
Authorization: Bearer eyJ...
```

Flow:

``` text
HTTP Request
    ↓
BearerTokenAuthenticationFilter
    ↓
extract bearer token
    ↓
BearerTokenAuthenticationToken
    ↓
AuthenticationManager
    ↓
JwtAuthenticationProvider
    ↓
JwtDecoder
    │
    ├─ parse token
    ├─ select verification key
    ├─ verify signature
    └─ run validators
    ↓
JwtAuthenticationConverter
    │
    └─ convert scopes/claims to authorities
    ↓
JwtAuthenticationToken
    ↓
SecurityContext
```

After that, authorization rules run.

For example:

``` java
@PreAuthorize("hasAuthority('SCOPE_payments.read')")
```

The controller should not be responsible for manually decoding and
cryptographically verifying every token.

That belongs in the security infrastructure.

------------------------------------------------------------------------

# 41. Spring Resource Server --- Opaque Internals

Flow:

``` text
HTTP Request
    ↓
BearerTokenAuthenticationFilter
    ↓
BearerTokenAuthenticationToken
    ↓
AuthenticationManager
    ↓
OpaqueTokenAuthenticationProvider
    ↓
OpaqueTokenIntrospector
    ↓
Authorization Server /introspect
    ↓
active + attributes + scopes
    ↓
Authentication
    ↓
SecurityContext
```

The high-level filter is similar; the token-validation strategy differs.

------------------------------------------------------------------------

# 42. Spring `oauth2Login()` Internals

For OIDC login:

``` text
Browser
   ↓
protected URL
   ↓
Spring Security
   ↓
OAuth2AuthorizationRequestRedirectFilter
   │
   ├─ construct authorization request
   ├─ state
   ├─ OIDC nonce
   └─ PKCE where configured/applicable
   ↓
Authorization Server
   ↓
user login
   ↓
callback with code
   ↓
OAuth2LoginAuthenticationFilter
   ↓
code exchange
   ↓
Access Token + ID Token
   ↓
OIDC validation/user loading
   ↓
OidcUser
   ↓
OAuth2AuthenticationToken
   ↓
SecurityContext
   ↓
typically persisted in HttpSession
```

This explains why a Spring web application using OIDC can still use a
normal `JSESSIONID` for subsequent browser requests.

OIDC handled the login; the Spring application can then maintain its own
local session.

------------------------------------------------------------------------

# 43. Scopes, Roles, Permissions and Claims

## Claim

Any token assertion:

``` text
sub
aud
iss
department
scope
tenant
```

## Scope

OAuth authorization vocabulary.

Examples:

``` text
accounts.read
payments.create
```

Think:

``` text
"What API capability was granted to this token/client?"
```

## Role

Business/security classification.

``` text
CUSTOMER
TELLER
BRANCH_MANAGER
ADMIN
```

## Permission

Fine-grained application capability.

``` text
PAYMENT_APPROVE
ACCOUNT_VIEW
CUSTOMER_EDIT
```

Roles may aggregate permissions:

``` text
BRANCH_MANAGER
     ├─ ACCOUNT_VIEW
     ├─ CUSTOMER_VIEW
     └─ PAYMENT_APPROVE
```

------------------------------------------------------------------------

# 44. Spring `GrantedAuthority`

Spring authorization works around `GrantedAuthority`.

Examples:

``` text
SCOPE_payments.read
ROLE_ADMIN
PAYMENT_APPROVE
```

OAuth scope:

``` text
payments.read
      ↓
SCOPE_payments.read
```

Then:

``` java
hasAuthority("SCOPE_payments.read")
```

Role:

``` java
hasRole("ADMIN")
```

conventionally checks:

``` text
ROLE_ADMIN
```

So:

``` java
hasRole("ADMIN")
```

and:

``` java
hasAuthority("ADMIN")
```

are not the same check.

------------------------------------------------------------------------

# 45. RBAC

Role-Based Access Control:

``` text
Principal
    ↓
Role
    ↓
Permissions
```

Example:

``` text
Bob
 ↓
BRANCH_MANAGER
 ↓
ACCOUNT_VIEW
PAYMENT_APPROVE
REPORT_VIEW
```

RBAC is useful when business authorization naturally follows
organizational roles.

But pure RBAC may be insufficient.

------------------------------------------------------------------------

# 46. ABAC

Attribute-Based Access Control evaluates attributes/context.

Suppose:

``` text
Bob.role = BRANCH_MANAGER
Bob.branch = HYD

Payment.branch = MUM
Payment.amount = ₹300000

Bob.approvalLimit = ₹500000
```

RBAC alone:

``` text
BRANCH_MANAGER
     ↓
PAYMENT_APPROVE
     ↓
ALLOW?
```

But ABAC can say:

``` text
role == BRANCH_MANAGER
AND
user.branch == payment.branch
AND
payment.amount <= approvalLimit
```

Branch fails:

``` text
HYD != MUM
```

Therefore deny.

------------------------------------------------------------------------

# 47. Object-Level Authorization

This is crucial for API security.

Token contains:

``` text
SCOPE_accounts.read
```

Request:

``` http
GET /accounts/12345
```

Scope answers:

``` text
May this token use account-reading functionality?
```

It does not answer:

``` text
May Alice read account 12345?
```

Correct authorization:

``` text
JWT valid?
   ↓
accounts.read?
   ↓
load account 12345
   ↓
owner/authorized party == Alice?
   ↓
tenant/business constraints?
   ↓
ALLOW
```

Spring example:

``` java
@PreAuthorize("""
    hasAuthority('SCOPE_accounts.read')
    && @accountSecurity.canRead(authentication, #accountId)
""")
```

This combines protocol-level authority and domain-level authority.

------------------------------------------------------------------------

# 48. Gateway vs Service Authorization

Gateway can perform:

``` text
token syntactically/authentically valid?
issuer?
audience?
coarse scope?
route policy?
```

But Gateway often does not know:

``` text
Who owns account 12345?
What is Alice's transfer limit?
Does this payment belong to Alice's branch?
```

Therefore:

``` text
Gateway
   ↓
coarse authorization

Business Service
   ↓
domain/object authorization
```

Do not centralize every business authorization rule at the gateway.

------------------------------------------------------------------------

# 49. Token Propagation Across Microservices

Scenario:

``` text
Alice
 │
 │ T1
 ▼
Service A
 │
 ▼
Service B
```

The wrong first question:

``` text
"Should I just forward T1?"
```

Correct first question:

``` text
"Whose authority should Service B see?"
```

Possibilities:

``` text
Alice
Service A
Alice delegated through Service A
Alice + authenticated workload A
```

------------------------------------------------------------------------

# 50. Token Relay

``` text
Alice ── T1 ──► A ── T1 ──► B
```

This is valid only when T1 is intended/acceptable for B according to the
authorization architecture.

Example problem:

``` json
"aud": "service-a"
```

A forwards it to B.

B sees a valid signature.

Should B accept it?

No, not merely because the signature is valid.

Audience says the token was intended for A.

This is why audience validation matters in microservices.

------------------------------------------------------------------------

# 51. Client Credentials for Downstream Calls

Suppose B does not care about Alice.

It only needs to know:

``` text
Service A is authorized to invoke operation X.
```

Then A can obtain its own token:

``` text
A
 │ client_credentials
 ▼
Authorization Server
 │
 ▼
T2 representing A
 │
 ▼
B
```

Now B sees workload/client authority rather than Alice's delegated
identity.

This is often cleaner for background/internal service operations.

------------------------------------------------------------------------

# 52. Token Exchange / On-Behalf-Of

Suppose B does need Alice's delegated identity, but T1 was issued for A.

Instead of blindly forwarding T1:

``` text
Alice
 │ T1, aud=A
 ▼
A
 │
 │ exchange T1
 ▼
Authorization Server
 │
 │ evaluate delegation
 │ issue T2
 ▼
A
 │ T2, aud=B
 ▼
B
```

T2 can have:

``` text
aud = B
narrower scopes
Alice subject/delegation context
calling client context
short lifetime
```

This lets the Authorization Server remain the authority deciding whether
A may act on Alice's behalf toward B.

------------------------------------------------------------------------

# 53. mTLS + OAuth Together

These mechanisms answer different questions.

``` text
A ── mTLS ──► B
```

mTLS tells B:

``` text
"The TLS peer is workload A."
```

OAuth token can tell B:

``` text
"This call carries Alice's delegated payments.read authority."
```

Combined:

``` text
Transport/workload identity
       +
Delegated API authority
       ↓
stronger service-to-service trust model
```

This is common in zero-trust-style architectures.

------------------------------------------------------------------------

# 54. Logout and the Four Independent Lifecycles

A user may have all of these:

``` text
1. Local Spring HttpSession
2. Authorization Server / IdP SSO session
3. Access Token
4. Refresh Token
```

They are not one object.

## Local logout

``` text
POST /logout
   ↓
clear SecurityContext
   ↓
invalidate HttpSession
   ↓
remove local JSESSIONID
```

But the IdP may still have an SSO session.

User starts login again:

``` text
Spring → IdP
          ↓
IdP already knows user
          ↓
login may complete without password
```

## JWT lifetime

Suppose:

``` text
Access Token exp = 11:00
Local logout = 10:30
```

A stolen copy can potentially still be used at 10:40 because local
Spring logout did not change the signed JWT.

Therefore:

``` text
Local logout
   ≠ IdP logout
   ≠ Access Token revocation
   ≠ Refresh Token revocation
```

Architectures must define each lifecycle deliberately.

------------------------------------------------------------------------

# 55. Complete Browser-to-Microservice Example

Consider:

``` text
React Browser
    │
    ▼
Spring BFF
    │
    ▼
Authorization Server
    │
    ▼
Order API
    │
    ▼
Payment API
```

## Step 1 --- Browser opens BFF

``` text
GET /orders
```

No local session.

Spring redirects to Authorization Server.

## Step 2 --- OIDC login

Authorization request contains relevant parameters such as:

``` text
client_id
redirect_uri
response_type=code
scope=openid ...
state
nonce
PKCE challenge where used
```

## Step 3 --- User authenticates

Authorization Server authenticates Alice.

## Step 4 --- Authorization code returned

Browser returns to BFF:

``` text
/callback?code=ABC&state=S1
```

Spring verifies transaction state.

## Step 5 --- Code exchange

BFF sends:

``` text
code=ABC
PKCE verifier where applicable
client authentication if applicable
```

Token endpoint returns:

``` text
ID Token
Access Token
possibly Refresh Token
```

## Step 6 --- Spring establishes local login

Spring validates OIDC response/ID Token and creates:

``` text
OidcUser
OAuth2AuthenticationToken
SecurityContext
HttpSession
```

Browser receives:

``` text
JSESSIONID
```

## Step 7 --- Browser calls BFF again

``` http
GET /orders
Cookie: JSESSIONID=...
```

The browser does not necessarily need to receive the backend Access
Token.

## Step 8 --- BFF calls Order API

BFF uses an appropriate Access Token:

``` http
Authorization: Bearer T1
```

Order API acts as Resource Server.

## Step 9 --- Order API validates JWT

``` text
BearerTokenAuthenticationFilter
      ↓
JwtAuthenticationProvider
      ↓
JwtDecoder
      ↓
cached JWK public key
      ↓
signature + issuer + time + configured audience
      ↓
Authentication
```

## Step 10 --- Authorization

Order API checks:

``` text
SCOPE_orders.read
```

and object/domain rules.

## Step 11 --- Order calls Payment

Now architecture must decide:

``` text
Does Payment need Alice?
Does Payment need only Order Service?
Is T1 intended for Payment?
```

Possible outcomes:

``` text
Token Relay
Client Credentials
Token Exchange/OBO
```

This is why "just forward the JWT" is not an architectural rule.

------------------------------------------------------------------------

# 56. Common Misconceptions

## Misconception 1

> JWT is encrypted.

Usually false.

JWT claims are commonly readable. Signature protects
integrity/authenticity.

## Misconception 2

> Resource Server downloads JWKS for every request.

False in normal operation.

It validates every JWT request, but verification keys are normally
cached.

## Misconception 3

> Opaque token is not validated every request.

Normally it is introspected for each request unless caching is
explicitly added.

## Misconception 4

> If JWT signature is valid, request is authorized.

False.

You still need issuer, time, audience/policy, scopes and domain
authorization.

## Misconception 5

> OAuth authenticates the user.

OAuth is primarily authorization. OIDC adds standardized user
authentication.

## Misconception 6

> ID Token should be sent to every API.

No. Access Tokens are intended for Resource Servers.

## Misconception 7

> `Access-Control-Allow-Credentials` prevents the controller from
> executing.

Not necessarily. CORS is browser response-access enforcement.

## Misconception 8

> HttpOnly means browser cannot send the cookie.

False.

It means JavaScript cannot normally read it; browser can still send it.

## Misconception 9

> mTLS and OAuth are alternatives.

Not necessarily.

mTLS can authenticate workload/connection while OAuth expresses API
authority.

## Misconception 10

> Every downstream service should accept the original user token.

False.

Audience and delegation semantics matter.

------------------------------------------------------------------------

# 57. Architect-Level Decision Framework

When designing an API, ask in this order.

## 1. How is transport secured?

``` text
HTTPS
mTLS where workload identity is needed
```

## 2. Who is the caller?

``` text
human user?
browser session?
service?
partner?
```

## 3. Which authentication protocol?

``` text
OIDC for interactive federated login
mTLS for workload/partner certificate identity
client authentication for machine clients
```

## 4. Which API credential?

``` text
JWT Access Token?
Opaque Access Token?
API key for simpler case?
```

## 5. How will Resource Server validate it?

JWT:

``` text
signature + issuer + time + audience/policy
```

Opaque:

``` text
introspection + active state + metadata
```

## 6. What authorization layers exist?

``` text
scope
role
permission
ownership
tenant
transaction limits
business state
```

## 7. What happens downstream?

``` text
relay?
client credentials?
token exchange?
mTLS?
```

## 8. What is the revocation/logout model?

``` text
Access Token lifetime
Refresh Token rotation
IdP session
local application session
emergency revocation
```

This produces a security architecture rather than a collection of
annotations.

------------------------------------------------------------------------

# 58. Spring Configuration Examples

## JWT Resource Server

``` java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .requestMatchers(HttpMethod.GET, "/payments/**")
                .hasAuthority("SCOPE_payments.read")
            .requestMatchers(HttpMethod.POST, "/payments/**")
                .hasAuthority("SCOPE_payments.create")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt());

    return http.build();
}
```

Typical issuer configuration:

``` yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

Conceptually Spring uses issuer metadata/JWK configuration to validate
JWTs.

If your architecture requires explicit audience validation, configure an
appropriate validator rather than assuming signature validation alone
covers audience policy.

## Opaque Resource Server

``` java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.opaqueToken());

    return http.build();
}
```

Configuration conceptually includes:

``` yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/introspect
          client-id: payment-api
          client-secret: ${INTROSPECTION_SECRET}
```

The Resource Server authenticates itself to introspection and obtains
token state.

## Method authorization

``` java
@PreAuthorize("hasAuthority('SCOPE_accounts.read')")
public Account getAccount(String id) {
    ...
}
```

Better when object-level authorization is required:

``` java
@PreAuthorize("""
    hasAuthority('SCOPE_accounts.read')
    && @accountSecurity.canRead(authentication, #id)
""")
public Account getAccount(String id) {
    ...
}
```

## OIDC login

Conceptually:

``` java
http
    .authorizeHttpRequests(auth -> auth
        .anyRequest().authenticated()
    )
    .oauth2Login();
```

This makes the Spring application an OAuth/OIDC Client for interactive
login, not a Resource Server merely by itself.

------------------------------------------------------------------------

# 59. Interview Questions and Answers

## Q1. Does Resource Server validate JWT on every request?

**Yes**, for every stateless bearer-token request. It normally performs
validation locally using cached public verification keys. It does not
normally call the Authorization Server for every request.

## Q2. Does Resource Server validate opaque token on every request?

Normally yes, through introspection. Explicit caching can reduce calls
but delays revocation visibility.

## Q3. Why does JWT scale well?

Because each Resource Server can independently verify signed tokens
using public keys rather than synchronously asking the issuer about each
request.

## Q4. What is the disadvantage?

Changes such as central account disablement/revocation are not
automatically visible to an offline verifier until token expiry or
another revocation mechanism intervenes.

## Q5. Why does opaque token make revocation easier?

Because the Resource Server asks the central token authority about
current token state.

## Q6. What is JWKS?

A published set of public verification keys. JWT's `kid` helps select
the correct key.

## Q7. Does `kid` prove anything by itself?

No. It is a key identifier used to locate candidate verification
material.

## Q8. What is PKCE?

A mechanism binding authorization-code redemption to the client instance
that initiated the authorization request using a verifier/challenge
pair.

## Q9. `state` vs PKCE?

`state` correlates/protects the authorization response transaction. PKCE
protects code redemption.

## Q10. `nonce` vs `state`?

`nonce` is associated with OIDC authentication/ID Token binding. `state`
is authorization transaction correlation.

## Q11. Access Token vs ID Token?

Access Token is presented to a Resource Server. ID Token is an
authentication assertion intended for the OIDC Client.

## Q12. Why not forward ID Token to API?

Because it is not the API authorization credential; the API should
receive an Access Token intended for it.

## Q13. What happens when TLS terminates at Ingress?

The Ingress owns the external TLS session. Backend Java does not
automatically receive the original client X.509 servlet attribute.
Trusted certificate identity must be securely propagated or TLS must be
passed through/re-established according to the architecture.

## Q14. Can React use session cookies?

Yes. Cookies are browser functionality. `HttpOnly` prevents JavaScript
reading but does not stop the browser sending them.

## Q15. What does `Access-Control-Allow-Credentials: true` do?

It participates in browser CORS policy for exposing credentialed
cross-origin responses. Its absence does not necessarily prevent
server-side processing.

## Q16. Why are scopes not enough?

Scope is coarse API capability. The application may still need
ownership, tenant, amount limits, role, branch, resource state and other
domain rules.

## Q17. Role vs scope?

Role describes a business/security classification. Scope represents
OAuth-delegated API capability.

## Q18. When should Service A use client credentials to call B?

When B should authorize Service A's own workload/client identity rather
than the original human user's delegated identity.

## Q19. When use token exchange/OBO?

When B needs delegated user context but should receive a B-specific
token/audience rather than blindly accepting A's token.

## Q20. Is logout equal to token revocation?

No. Local session, IdP SSO session, Access Token and Refresh Token have
separate lifecycles.

------------------------------------------------------------------------

# 60. Final Memory Map

``` text
                           SECURITY

Transport
   │
   ├── TLS
   │     └── encryption + server authentication
   │
   └── mTLS
         └── TLS + client/workload authentication


Browser Authentication
   │
   ├── Session/Cookie
   │     ├── HttpOnly
   │     ├── Secure
   │     └── SameSite
   │
   └── OIDC
         └── ID Token + standardized authentication


API Authorization
   │
   └── OAuth 2.0
         │
         ├── Authorization Code + PKCE
         ├── Client Credentials
         │
         └── Access Token
               │
               ├── JWT
               │     ├── signed
               │     ├── validate every request
               │     ├── local verification
               │     └── cached JWKS/public keys
               │
               └── Opaque
                     ├── reference credential
                     ├── validate every request
                     └── introspection


Spring Security
   │
   ├── oauth2Login()
   │     └── login user into Spring app
   │
   ├── oauth2Client()
   │     └── outbound OAuth calls
   │
   └── oauth2ResourceServer()
         ├── JwtDecoder
         └── OpaqueTokenIntrospector


Authorization
   │
   ├── Scope
   ├── Role
   ├── Permission
   ├── RBAC
   ├── ABAC
   └── Object/domain authorization


Microservices
   │
   ├── Token Relay
   ├── Client Credentials
   ├── Token Exchange / OBO
   └── mTLS + OAuth


Lifecycle
   │
   ├── Local Session
   ├── IdP SSO Session
   ├── Access Token
   └── Refresh Token
         └── all must be reasoned about separately
```

------------------------------------------------------------------------

# The Core Principle

If you remember only one architecture diagram, remember this:

``` text
REQUEST
   │
   ▼
TLS / mTLS
"Is the communication channel trusted?"
   │
   ▼
Authentication / Token Validation
"Who/what is calling, and is the credential valid?"
   │
   ▼
OAuth Authorization
"Was this API capability granted?"
   │
   ▼
Business Authorization
"May this principal perform THIS operation
on THIS resource in THIS context?"
   │
   ▼
BUSINESS LOGIC
```

That separation is the foundation for understanding OAuth, OIDC, Spring
Security and enterprise API security correctly.
