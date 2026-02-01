# Certificates and PKI

Certificates prove a server's identity. The **Public Key Infrastructure (PKI)** is the trust hierarchy that makes this verification possible.

## What's in a Certificate

```
X.509 Certificate Structure:
┌─────────────────────────────────────────────────────────────────────┐
│  Version: 3 (X.509v3)                                               │
│  Serial Number: 04:00:00:00:00:01:15:4B:5A:C3:94                     │
│  Signature Algorithm: sha256WithRSAEncryption                       │
│                                                                     │
│  Issuer: CN=DigiCert Global CA, O=DigiCert Inc, C=US                │
│  Validity:                                                          │
│      Not Before: Jan 15 00:00:00 2024 GMT                           │
│      Not After:  Jan 14 23:59:59 2025 GMT                           │
│  Subject: CN=www.example.com, O=Example Inc, C=US                   │
│                                                                     │
│  Subject Public Key Info:                                           │
│      Public Key Algorithm: rsaEncryption                            │
│      RSA Public Key: (2048 bit)                                     │
│          Modulus: 00:c3:9b:...                                      │
│          Exponent: 65537                                            │
│                                                                     │
│  X509v3 Extensions:                                                 │
│      Subject Alternative Name:                                      │
│          DNS:www.example.com, DNS:example.com                       │
│      Key Usage: Digital Signature, Key Encipherment                 │
│      Extended Key Usage: TLS Web Server Authentication              │
│                                                                     │
│  Signature: 3c:b3:4e:...                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Fields

| Field | Purpose |
|-------|---------|
| Subject | Who the certificate identifies |
| Issuer | Who signed (issued) the certificate |
| Validity | When certificate is valid |
| Public Key | Server's public key for encryption |
| Subject Alt Names | Additional valid hostnames |
| Signature | CA's signature verifying the certificate |

## Certificate Chain

Certificates form a chain of trust:

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Certificate Chain                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────┐                            │
│  │          Root CA Certificate        │ ← Trusted by OS/browser   │
│  │  Issuer: DigiCert Root CA           │   (pre-installed)         │
│  │  Subject: DigiCert Root CA          │   Self-signed             │
│  └─────────────────┬───────────────────┘                            │
│                    │ Signs                                          │
│  ┌─────────────────▼───────────────────┐                            │
│  │     Intermediate CA Certificate     │ ← Server sends this       │
│  │  Issuer: DigiCert Root CA           │                            │
│  │  Subject: DigiCert Global CA        │                            │
│  └─────────────────┬───────────────────┘                            │
│                    │ Signs                                          │
│  ┌─────────────────▼───────────────────┐                            │
│  │       End-Entity Certificate        │ ← Server's certificate    │
│  │  Issuer: DigiCert Global CA         │                            │
│  │  Subject: www.example.com           │                            │
│  └─────────────────────────────────────┘                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Validation:
1. Server sends leaf + intermediate(s)
2. Client finds trusted root CA
3. Verifies each signature up the chain
4. Checks validity dates, revocation, hostname
5. Trust established!
```

## Certificate Validation

What clients check:

```
1. Chain of Trust
   Each certificate signed by issuer's key
   Chain leads to trusted root CA

2. Validity Period
   Current date within Not Before / Not After

3. Hostname Match
   Requested hostname in Subject CN or SAN
   www.example.com matches *.example.com (wildcard)

4. Revocation Status
   Certificate not revoked (CRL or OCSP)

5. Key Usage
   Certificate allowed for TLS server authentication

6. Cryptographic Verification
   Signatures mathematically valid
   Key sizes acceptable
```

## Certificate Types

### Domain Validation (DV)

```
Proves: Control of domain
Verification: Email, DNS, or HTTP challenge
Trust level: Low (only domain ownership)
Example: Let's Encrypt certificates
```

### Organization Validation (OV)

```
Proves: Domain control + organization exists
Verification: Legal documents, phone calls
Trust level: Medium
Example: Business websites
```

### Extended Validation (EV)

```
Proves: Domain + organization + legal verification
Verification: Extensive background checks
Trust level: High
Example: Banks, financial institutions
Note: Browsers no longer show green bar
```

## Getting Certificates

### Let's Encrypt (Free, Automated)

```bash
# Using certbot
$ sudo certbot certonly --webroot -w /var/www/html -d example.com

# Auto-renewal
$ sudo certbot renew

# Certificates at:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

### Commercial CAs

1. Generate CSR (Certificate Signing Request)
2. Submit to CA with payment
3. Complete validation
4. Receive certificate

```bash
# Generate private key and CSR
$ openssl req -new -newkey rsa:2048 -nodes \
    -keyout server.key -out server.csr

# Submit server.csr to CA
# Receive server.crt back
```

## Certificate Revocation

When certificates need to be invalidated:

### CRL (Certificate Revocation List)

```
CA publishes list of revoked certificates.
Client downloads CRL, checks if cert is listed.

Problems:
  - CRLs can be large
  - Caching means delayed revocation detection
  - Download can be slow
```

### OCSP (Online Certificate Status Protocol)

```
Client asks CA: "Is this certificate revoked?"
CA responds: "Valid" or "Revoked"

Better than CRL but:
  - Latency for each connection
  - Privacy (CA sees what sites you visit)
```

### OCSP Stapling

```
Server fetches OCSP response periodically.
Server "staples" response to TLS handshake.
Client gets proof without contacting CA.

Best practice: Enable OCSP stapling on your server.
```

## Common Issues

### Missing Intermediate

```
Problem:
  Server sends only leaf certificate
  Client can't build chain to root
  "Certificate not trusted" error

Solution:
  Configure server to send full chain:
  ssl_certificate /path/to/fullchain.pem;
  (includes leaf + intermediates)
```

### Expired Certificate

```
Problem:
  Certificate validity period ended
  Browsers show security warning

Solution:
  Renew certificate before expiration
  Set up automated renewal (Let's Encrypt)
  Monitor certificate expiration
```

### Hostname Mismatch

```
Problem:
  Request to example.com
  Certificate for www.example.com
  Names don't match

Solution:
  Include all domains in SAN
  Use wildcard (*.example.com) if appropriate
  Redirect to canonical hostname
```

## Summary

Certificate/PKI key concepts:

| Component | Purpose |
|-----------|---------|
| Certificate | Binds public key to identity |
| Root CA | Trusted anchor (pre-installed) |
| Intermediate CA | Bridges root to end-entity |
| Chain | Path from leaf to trusted root |
| CRL/OCSP | Revocation checking |
| SAN | Multiple hostnames in one cert |

Best practices:
- Use certificates from trusted CAs
- Include full certificate chain
- Enable OCSP stapling
- Monitor expiration dates
- Use strong key sizes (RSA 2048+ or ECDSA P-256+)
