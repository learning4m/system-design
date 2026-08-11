<a id="top"></a>

# Cryptography & OpenSSL Fundamentals

> A practical, interview-friendly guide to asymmetric cryptography, RSA keys, private-key formats, X.509 certificates, certificate chains, CSRs, TLS, and common OpenSSL commands.

## Table of Contents

1. [Asymmetric Cryptography](#1-asymmetric-cryptography)
2. [Generating RSA Key Pairs](#2-generating-rsa-key-pairs)
3. [Inspecting Keys](#3-inspecting-keys)
4. [Private-Key Encryption and PKCS Formats](#4-private-key-encryption-and-pkcs-formats)
5. [X.509 Certificates](#5-x509-certificates)
6. [Certificate Authorities and Certificate Chains](#6-certificate-authorities-and-certificate-chains)
7. [Certificate Signing Requests](#7-certificate-signing-requests-csr)
8. [Certificate Issuance Flow](#8-certificate-issuance-flow)
9. [How Certificates Participate in TLS](#9-how-certificates-participate-in-tls)
10. [Inspecting Certificates and CSRs](#10-inspecting-certificates-and-csrs)
11. [OpenSSL s_client](#11-openssl-s_client)
12. [PEM, DER and PKCS#12/PFX](#12-pem-der-and-pkcs12pfx)
13. [Command Cheat Sheet](#13-command-cheat-sheet)
14. [Important Corrections / Modern OpenSSL Notes](#14-important-corrections--modern-openssl-notes)
15. [Interview Mental Model](#15-interview-mental-model)

---

# 1. Asymmetric Cryptography

Asymmetric cryptography uses a **key pair**:

- **Private key** — kept secret by the owner.
- **Public key** — may be distributed freely.

The keys are mathematically related, but possession of the public key should not make it computationally feasible to derive the private key.

```mermaid
flowchart LR
    A[Asymmetric Key Pair] --> B[Private Key]
    A --> C[Public Key]
    B --> D[Keep Secret]
    C --> E[Share Freely]
```

## Major operations

### Encryption

Conceptually, with an RSA encryption-capable key:

```text
Public Key  -> Encrypt
Private Key -> Decrypt
```

```mermaid
sequenceDiagram
    participant Alice as Sender
    participant Bob as Receiver
    Alice->>Alice: Obtain Bob's public key
    Alice->>Bob: Ciphertext encrypted for Bob
    Bob->>Bob: Decrypt using Bob's private key
```

In real TLS, however, bulk application data is **not encrypted directly with RSA**. Modern TLS establishes symmetric traffic keys and uses efficient symmetric authenticated encryption for the actual data.

### Digital signatures

The owner signs using the **private key**, while others verify using the corresponding **public key**.

```mermaid
flowchart LR
    M[Message] --> H[Hash]
    H --> S[Sign using Private Key]
    S --> SIG[Digital Signature]
    M --> V[Verifier]
    SIG --> V
    PK[Public Key] --> V
    V --> R{Signature valid?}
```

A signature provides evidence of integrity and possession of the private key. It does **not** encrypt the message.

### Key agreement / exchange

Algorithms such as Diffie-Hellman and ECDH allow parties to derive shared secret material over an untrusted network.

| Algorithm | Encryption | Signature | Key Agreement / Exchange |
|---|---:|---:|---:|
| RSA | Yes* | Yes | Historically usable for RSA key transport* |
| DSA | No | Yes | No |
| DH | No | No | Yes |
| ECDSA | No | Yes | No |
| ECDH | No | No | Yes |

`*` Modern TLS 1.3 does not use RSA key transport; RSA certificates can still be used for authentication/signatures while ephemeral (EC)DHE establishes shared secrets.

**Key idea:** not every asymmetric algorithm supports every operation.


[↑ Back to Table of Contents](#table-of-contents)

---

# 2. Generating RSA Key Pairs

## Legacy/familiar command

```bash
openssl genrsa -out rsa-2048.pem 2048
```

`genrsa` is still familiar from older material, but modern OpenSSL recommends the more general `genpkey` interface.

## Modern command

```bash
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:2048 \
  -out rsa-2048.pem
```

For a larger RSA key:

```bash
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:4096 \
  -out rsa-4096.pem
```

The resulting private-key object contains the private parameters from which the public key can be obtained. Do **not** interpret this as two independent PEM keys simply concatenated in the file.

## Extract the public key

```bash
openssl pkey \
  -in rsa-2048.pem \
  -pubout \
  -out rsa-2048-public.pem
```

```mermaid
flowchart LR
    G[Generate RSA Private Key] --> PR[rsa-2048.pem]
    PR -->|openssl pkey -pubout| PU[rsa-2048-public.pem]
    PR --> X[Keep Secret]
    PU --> Y[Can Be Distributed]
```

> For security-sensitive production systems, use an organization-approved key size, algorithm, provider/HSM/KMS, and key-management policy rather than copying example commands blindly.


[↑ Back to Table of Contents](#table-of-contents)

---

# 3. Inspecting Keys

`openssl pkey` is the generic command for inspecting many private/public key types.

## Human-readable private-key information

```bash
openssl pkey -in rsa-2048.pem -text -noout
```

For RSA-specific inspection, the older RSA command is also available:

```bash
openssl rsa -in rsa-2048.pem -text -noout
```

## Print only the public key

```bash
openssl pkey -in rsa-2048.pem -pubout
```

## RSA modulus

For RSA keys:

```bash
openssl rsa -in rsa-2048.pem -modulus -noout
```

The modulus is one of the RSA public parameters. It is **not** itself the complete public key.


[↑ Back to Table of Contents](#table-of-contents)

---

# 4. Private-Key Encryption and PKCS Formats

A PEM file is commonly Base64-armored text surrounding an encoded cryptographic structure.

An unencrypted private key may look like:

```text
-----BEGIN PRIVATE KEY-----
(base64-encoded data)
-----END PRIVATE KEY-----
```

or, for a traditional RSA encoding:

```text
-----BEGIN RSA PRIVATE KEY-----
(base64-encoded data)
-----END RSA PRIVATE KEY-----
```

**Base64 is encoding, not encryption.** Anyone who obtains an unencrypted private-key file can potentially use that private key.

## Protecting a private key with a passphrase

The passphrase is processed through a key-derivation mechanism to protect the private key using symmetric encryption.

```mermaid
flowchart LR
    P[Passphrase] --> KDF[Password-based KDF]
    KDF --> K[Encryption Key]
    PR[Private Key] --> ENC[Symmetric Encryption]
    K --> ENC
    ENC --> F[Encrypted Private-Key File]
```

## PKCS#8 — preferred generic private-key format

Convert a private key to encrypted PKCS#8:

```bash
openssl pkcs8 \
  -topk8 \
  -in rsa-2048.pem \
  -out rsa-2048-pkcs8.pem
```

OpenSSL prompts for an output passphrase.

An encrypted PKCS#8 PEM commonly looks like:

```text
-----BEGIN ENCRYPTED PRIVATE KEY-----
...
-----END ENCRYPTED PRIVATE KEY-----
```

Create an unencrypted PKCS#8 representation:

```bash
openssl pkcs8 \
  -topk8 \
  -nocrypt \
  -in rsa-2048.pem \
  -out rsa-2048-pkcs8-unencrypted.pem
```

## Traditional / PKCS#1 RSA private key

PKCS#1 is RSA-specific. Modern applications generally favor generic PKCS#8 where possible.

A traditional encrypted RSA PEM may include legacy-style encryption headers, depending on the OpenSSL command/version used.

**Rule of thumb:**

```text
PKCS#1 -> RSA-specific key structures
PKCS#8 -> Generic private-key container/format
```


[↑ Back to Table of Contents](#table-of-contents)

---

# 5. X.509 Certificates

An X.509 certificate binds an **identity/name and related attributes** to a **public key**, with the binding authenticated by a digital signature.

Think of it as:

```text
Identity information
       +
Public key
       +
Validity / policy / extensions
       +
Issuer's digital signature
       =
X.509 Certificate
```

```mermaid
flowchart TD
    C[X.509 Certificate]
    C --> S[Subject]
    C --> SAN[Subject Alternative Name]
    C --> PK[Subject Public Key]
    C --> I[Issuer]
    C --> V[Validity Period]
    C --> E[Extensions / Constraints / Usage]
    C --> SIG[Issuer Signature]
```

Important certificate fields/extensions include:

- Subject
- Subject Alternative Name (SAN)
- Issuer
- Public key / public-key algorithm
- Serial number
- Not Before / Not After validity
- Key Usage / Extended Key Usage
- Basic Constraints
- Signature algorithm and issuer signature

For modern HTTPS hostname verification, **SAN is particularly important**.

## Server authentication vs mutual TLS

Normal public HTTPS usually authenticates the server to the client.

```text
Client ----------------> Server
       verifies server certificate
```

With **mutual TLS (mTLS)**, both parties present credentials:

```text
Client <--------------> Server
certificate              certificate
+ private-key proof       + private-key proof
```


[↑ Back to Table of Contents](#table-of-contents)

---

# 6. Certificate Authorities and Certificate Chains

A **Certificate Authority (CA)** issues/signs certificates.

A public PKI normally uses a hierarchy rather than having one root directly sign every website certificate.

```mermaid
flowchart TD
    R[Root CA Certificate\nTrusted locally] -->|signs| I1[Intermediate CA Certificate]
    I1 -->|signs| I2[Intermediate CA Certificate]
    I2 -->|signs| L[Leaf / Server Certificate\nexample.com]
```

Each CA normally has its own key pair.

### Root CA

The root certificate is normally self-signed:

```text
Subject: Root CA
Issuer : Root CA
```

The crucial reason a client trusts it is **not merely because it is self-signed**. It is trusted because the root is configured as a trust anchor in the client's trust store.

### Intermediate CA

```text
Subject: Intermediate CA
Issuer : Root CA
```

### Leaf/server certificate

```text
Subject/SAN: example.com
Issuer     : Intermediate CA
```

## What the client verifies

Conceptually:

```mermaid
flowchart BT
    L[example.com Certificate] -->|verify issuer signature| I[Intermediate CA]
    I -->|verify issuer signature| R[Trusted Root CA]
    R --> T[Local Trust Store]
```

Actual path validation also checks validity, names, constraints, key usage, policies as applicable, and potentially revocation/status information.

Browsers, operating systems, JVMs, containers, and individual applications may use different trust stores.


[↑ Back to Table of Contents](#table-of-contents)

---

# 7. Certificate Signing Requests (CSR)

A server first generates a key pair and keeps its **private key private**.

It then creates a **Certificate Signing Request (CSR)**, normally PKCS#10.

The CSR contains, among other data:

- The public key
- Requested subject information
- Often requested extensions such as SAN
- A signature made using the applicant's private key

The private key itself is **not sent to the CA**.

```mermaid
flowchart LR
    KP[Server Key Pair] --> PR[Private Key]
    KP --> PU[Public Key]
    PU --> CSR[CSR]
    INFO[Subject / Requested Extensions] --> CSR
    PR -->|signs CSR| CSR
    CSR --> CA[Certificate Authority]
```

## Create a CSR using an existing key

```bash
openssl req \
  -new \
  -key rsa-2048.pem \
  -out server.csr
```

A CSR in PEM form looks like:

```text
-----BEGIN CERTIFICATE REQUEST-----
...
-----END CERTIFICATE REQUEST-----
```

## Inspect the CSR

```bash
openssl req \
  -in server.csr \
  -noout \
  -text
```

Verify its self-signature:

```bash
openssl req -in server.csr -verify -noout
```

Extract its public key:

```bash
openssl req \
  -in server.csr \
  -pubkey \
  -noout
```


[↑ Back to Table of Contents](#table-of-contents)

---

# 8. Certificate Issuance Flow

The high-level lifecycle is:

```mermaid
sequenceDiagram
    participant S as Server / Applicant
    participant CA as Certificate Authority
    participant C as Client

    S->>S: Generate private/public key pair
    S->>S: Create CSR containing public key
    S->>S: Sign CSR with private key
    S->>CA: Submit CSR
    CA->>CA: Validate request / identity / policy
    CA->>CA: Build certificate from approved data
    CA->>CA: Sign certificate with CA private key
    CA-->>S: Return signed certificate (+ chain as needed)
    C->>S: Connect using TLS
    S-->>C: Present leaf certificate + intermediates
    C->>C: Build and validate path to trusted root
```

The CA's signature is over the certificate data. It is **not** simply "signing the CSR and returning the CSR." The issued certificate is a distinct X.509 object.

## Self-signed certificate for testing

A convenient modern test command can generate a key and self-signed certificate together:

```bash
openssl req \
  -x509 \
  -newkey rsa:2048 \
  -keyout server-key.pem \
  -out server-cert.pem \
  -days 365 \
  -addext "subjectAltName=DNS:localhost"
```

For local testing you may choose `-noenc` if you intentionally do not want the generated key encrypted:

```bash
openssl req -x509 -newkey rsa:2048 -noenc \
  -keyout server-key.pem \
  -out server-cert.pem \
  -days 365 \
  -addext "subjectAltName=DNS:localhost"
```

Do not treat a self-signed test certificate as automatically trusted; clients must explicitly trust the relevant certificate/trust anchor.


[↑ Back to Table of Contents](#table-of-contents)

---

# 9. How Certificates Participate in TLS

A useful simplified picture is:

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: ClientHello
    S-->>C: ServerHello
    S-->>C: Certificate chain
    S-->>C: Proof using server private key
    C->>C: Validate certificate chain + hostname
    C->>S: Key-exchange contribution / Finished
    S-->>C: Finished
    Note over C,S: Both derive symmetric traffic keys
    C<<->>S: Encrypted authenticated application data
```

The exact TLS messages vary by TLS version and authentication mode, but this captures the architectural idea.

### The important separation

Certificates primarily solve **authentication / public-key binding**.

The handshake establishes **shared symmetric traffic secrets**.

Then application traffic uses fast symmetric authenticated encryption.

```mermaid
flowchart LR
    CERT[Certificate / PKI] --> AUTH[Authenticate Server]
    AUTH --> HS[TLS Handshake]
    KX[Ephemeral Key Agreement] --> HS
    HS --> SK[Symmetric Traffic Keys]
    SK --> DATA[Encrypted + Authenticated Application Data]
```

In TLS 1.3, key agreement normally uses ephemeral (EC)DHE. This also provides forward secrecy when appropriately configured.


[↑ Back to Table of Contents](#table-of-contents)

---

# 10. Inspecting Certificates and CSRs

## Inspect a certificate

```bash
openssl x509 \
  -in server-cert.pem \
  -noout \
  -text
```

Useful focused views:

```bash
openssl x509 -in server-cert.pem -noout -subject
openssl x509 -in server-cert.pem -noout -issuer
openssl x509 -in server-cert.pem -noout -dates
openssl x509 -in server-cert.pem -noout -serial
openssl x509 -in server-cert.pem -noout -pubkey
```

## Inspect a CSR

```bash
openssl req -in server.csr -noout -text
```

## Extract CSR public key

```bash
openssl req -in server.csr -pubkey -noout
```


[↑ Back to Table of Contents](#table-of-contents)

---

# 11. OpenSSL `s_client`

`s_client` is invaluable when debugging TLS.

```bash
openssl s_client -connect google.com:443
```

For SNI/hostname-oriented testing, explicitly supplying the server name is often clearer:

```bash
openssl s_client \
  -connect google.com:443 \
  -servername google.com
```

To request hostname verification as part of the command:

```bash
openssl s_client \
  -connect google.com:443 \
  -servername google.com \
  -verify_hostname google.com
```

It can help investigate:

- Presented certificate chain
- TLS version
- Cipher suite
- Verification errors
- SNI problems
- mTLS configuration
- Trust-chain problems


[↑ Back to Table of Contents](#table-of-contents)

---

# 12. PEM, DER and PKCS#12/PFX

One of the most important distinctions is between **encoding/container format** and **filename extension**.

## PEM

PEM is text-armored/Base64 representation with labels such as:

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

or

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

## DER

DER is a binary ASN.1 encoding.

```text
PEM = textual armor around encoded bytes
DER = binary encoding
```

Convert a PEM X.509 certificate to DER:

```bash
openssl x509 \
  -in server-cert.pem \
  -outform DER \
  -out server-cert.der
```

Convert DER back to PEM:

```bash
openssl x509 \
  -in server-cert.der \
  -inform DER \
  -out server-cert.pem
```

## PKCS#12 / PFX

PKCS#12 is a container that can hold combinations of:

- Private key
- Leaf certificate
- Intermediate/CA certificates

```mermaid
flowchart TD
    PFX[PKCS#12 / PFX Container]
    PFX --> K[Private Key]
    PFX --> L[Leaf Certificate]
    PFX --> C[CA / Intermediate Certificates]
```

Typical creation command:

```bash
openssl pkcs12 \
  -export \
  -inkey server-key.pem \
  -in server-cert.pem \
  -certfile chain.pem \
  -out server.p12
```

Inspect without exporting contents:

```bash
openssl pkcs12 -in server.p12 -info -noout
```

Extract certificates only:

```bash
openssl pkcs12 \
  -in server.p12 \
  -nokeys \
  -out certificates.pem
```

Extract private key (encrypted output by default/current options may vary):

```bash
openssl pkcs12 \
  -in server.p12 \
  -nocerts \
  -out private-key.pem
```

### Extensions do not guarantee formats

`.pem`, `.crt`, `.cer`, `.key`, `.p12`, and `.pfx` are naming conventions. Do not determine the underlying encoding solely from a filename.

For example, test whether a certificate parses as PEM:

```bash
openssl x509 -in file.crt -inform PEM -noout
```

or DER:

```bash
openssl x509 -in file.crt -inform DER -noout
```


[↑ Back to Table of Contents](#table-of-contents)

---

# 13. Command Cheat Sheet

| Goal | Command |
|---|---|
| Generate RSA private key | `openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out key.pem` |
| Extract public key | `openssl pkey -in key.pem -pubout -out public.pem` |
| Inspect private key | `openssl pkey -in key.pem -text -noout` |
| Create CSR | `openssl req -new -key key.pem -out server.csr` |
| Inspect CSR | `openssl req -in server.csr -text -noout` |
| Verify CSR signature | `openssl req -in server.csr -verify -noout` |
| Extract CSR public key | `openssl req -in server.csr -pubkey -noout` |
| Inspect certificate | `openssl x509 -in cert.pem -text -noout` |
| Show certificate subject | `openssl x509 -in cert.pem -subject -noout` |
| Show issuer | `openssl x509 -in cert.pem -issuer -noout` |
| Show validity | `openssl x509 -in cert.pem -dates -noout` |
| Convert key to PKCS#8 | `openssl pkcs8 -topk8 -in key.pem -out key-pkcs8.pem` |
| Connect to TLS server | `openssl s_client -connect host:443 -servername host` |
| Inspect PFX/P12 | `openssl pkcs12 -in file.p12 -info -noout` |


[↑ Back to Table of Contents](#table-of-contents)

---

# 14. Important Corrections / Modern OpenSSL Notes

These points refine some common introductory shorthand found in older notes/tutorials.

### 1. `genrsa` vs `genpkey`

`openssl genrsa` is an older RSA-specific interface. Modern OpenSSL documentation points toward `openssl genpkey` for general private-key/key-pair generation.

### 2. RSA is not the normal TLS key-exchange mechanism today

Older TLS versions could use RSA key transport. **TLS 1.3 removed RSA key transport**. RSA may still authenticate a server via signatures, while (EC)DHE establishes the shared secret.

### 3. A certificate does not contain a private key

```text
Certificate -> public key + identity/metadata + issuer signature
Private key -> held separately by its owner
```

A **PKCS#12/PFX container** may contain both a certificate and its private key, which is different from saying the X.509 certificate contains the private key.

### 4. CSR signature vs certificate signature

```text
CSR:
Applicant signs CSR using applicant's private key.

Certificate:
Issuer/CA signs certificate using CA's private key.
```

These signatures serve different purposes.

### 5. Root certificates are trusted because they are trust anchors

Self-signing alone does not create trust. A client trusts a root because that root is present/configured as an accepted trust anchor.

### 6. Base64 is not encryption

PEM's textual encoding does not protect a private key. Encryption/password protection must be applied separately.

### 7. TLS does not simply create "one session key"

Modern TLS derives multiple secrets/traffic keys for different directions and purposes. "Session key" is useful introductory shorthand, but **traffic keys/secrets** is more accurate.

### 8. MAC terminology in modern TLS

Older TLS cipher suites could use a separate MAC such as HMAC. Modern TLS commonly uses **AEAD** ciphers such as AES-GCM or ChaCha20-Poly1305, where confidentiality and integrity/authentication are integrated rather than represented as a separate application-data MAC key in the simplistic diagram.


[↑ Back to Table of Contents](#table-of-contents)

---

# 15. Interview Mental Model

If you remember only one flow, remember this:

```mermaid
flowchart TD
    A[1. Server generates key pair] --> B[2. Private key stays with server]
    A --> C[3. Public key goes into CSR]
    B --> D[4. Server signs CSR]
    C --> D
    D --> E[5. CSR submitted to CA]
    E --> F[6. CA validates request]
    F --> G[7. CA issues and signs X.509 certificate]
    G --> H[8. Server installs certificate + chain]
    H --> I[9. Client starts TLS connection]
    I --> J[10. Server presents certificate chain]
    J --> K[11. Client validates chain/name/validity/etc.]
    K --> L[12. TLS authenticates peers and establishes traffic secrets]
    L --> M[13. Symmetric AEAD protects application traffic]
```

## Certificate-chain mental model

```text
                  TRUST
                    ▲
                    │
              +-----------+
              |  Root CA  |   <- already trusted/configured
              +-----------+
                    │ signs
                    ▼
            +----------------+
            | Intermediate CA|
            +----------------+
                    │ signs
                    ▼
            +----------------+
            | Server / Leaf  |
            | Certificate    |
            +----------------+
                    │
                    ▼
                example.com
```

## Key ownership mental model

```text
SERVER
├── Private key        <-- SECRET; never sent in certificate/CSR
├── Public key
│      └── included in CSR
│              └── becomes public key in certificate
└── Certificate
       ├── identity / SAN
       ├── server public key
       └── CA signature

CA
├── CA private key     <-- signs issued certificates
└── CA certificate     <-- contains CA public key

CLIENT
└── Trust store
       └── trusted root CA certificates/public keys
```

That distinction explains a large part of TLS, mTLS, Java truststores/keystores, Kubernetes TLS secrets, ingress TLS termination, Kafka SSL, and service-to-service certificate authentication.


[↑ Back to Table of Contents](#table-of-contents)

---

## Next Topics

Natural follow-on topics are:

1. TLS 1.2 handshake vs TLS 1.3 handshake
2. [One-way TLS vs mTLS](#server-authentication-vs-mutual-tls)
3. Java `KeyStore` vs `TrustStore`
4. [PKCS#12 vs JKS](#12-pem-der-and-pkcs12pfx)
5. How Java/Spring Boot loads certificates
6. TLS termination at Kubernetes Ingress
7. Forwarding client identity after TLS termination
8. [Kafka SSL/mTLS and certificate DN (`CN`, `OU`, SAN) authorization](#server-authentication-vs-mutual-tls)
9. Certificate rotation and expiry
10. OCSP and CRLs

