# DNSSEC

**DNSSEC (Domain Name System Security Extensions)** adds cryptographic authentication to DNS. It allows resolvers to verify that DNS responses haven't been tampered with—protecting against attacks like cache poisoning.

## The Problem DNSSEC Solves

Traditional DNS has no authentication:

```
Without DNSSEC:

Client: "What's the IP for bank.com?"

Legitimate response:     OR    Attacker's response:
  bank.com → 1.2.3.4            bank.com → 6.6.6.6 (malicious)

How does client know which is real?
It can't! DNS responses are unsigned.

Attacks possible:
  - Cache poisoning (inject fake records)
  - Man-in-the-middle (intercept and modify)
  - Redirection to phishing sites
```

## How DNSSEC Works

DNSSEC adds digital signatures to DNS records:

```
With DNSSEC:

Zone operator:
  1. Generates signing keys
  2. Signs each record set
  3. Publishes signatures alongside records

Resolver:
  1. Receives record + signature
  2. Retrieves zone's public key
  3. Verifies signature
  4. If valid → Trust the record
  5. If invalid → Reject (SERVFAIL)

Attacker cannot forge valid signatures without private key.
```

## DNSSEC Record Types

### DNSKEY (Public Key)

```
Zone's public key for signature verification:

example.com.    3600    IN    DNSKEY    257 3 13 (
                                mdsswUyr3DPW132mOi8V9xESWE8jTo0d
                                xCjjnopKl+GqJxpVXckHAeF+KkxLbxIL
                                fDLUT0rAK9iUzy1L53eKGQ==
                              )

Fields:
  257 = Zone Signing Key (ZSK) or Key Signing Key (KSK)
  3   = Protocol (always 3)
  13  = Algorithm (13 = ECDSA P-256)
  Base64 = The public key
```

### RRSIG (Resource Record Signature)

```
Signature over a record set:

example.com.    3600    IN    A       93.184.216.34
example.com.    3600    IN    RRSIG   A 13 2 3600 (
                                20240215000000
                                20240201000000
                                12345 example.com.
                                oJB1W6WNGv+ldvQ3WDG0MQkg5IEhjRip
                                8WTrPYGv07h108dUKGMeDPKijVCHX3DD
                                Kdfb+v6oB9wfuh3DTJXUAfI= )

Fields:
  A         = Type being signed
  13        = Algorithm
  2         = Labels in name
  3600      = Original TTL
  Dates     = Signature validity period
  12345     = Key tag (identifies signing key)
  Base64    = The signature
```

### DS (Delegation Signer)

```
Links child zone to parent (chain of trust):

In .com zone:
  example.com.    86400    IN    DS    12345 13 2 (
                                  49FD46E6C4B45C55D4AC69CBD3CD3440
                                  9B20CAC6B08F4E7FAE3F2BDDBF1BB349 )

Fields:
  12345   = Key tag of child's KSK
  13      = Algorithm
  2       = Digest type (2 = SHA-256)
  Hex     = Hash of child's DNSKEY

Parent vouches for child's key.
Enables trust chain from root.
```

### NSEC/NSEC3 (Authenticated Denial)

```
Proves a name doesn't exist:

Query: nonexistent.example.com
Response: NXDOMAIN + NSEC record

NSEC proves there's no record between two names:
  aaa.example.com.    NSEC    zzz.example.com. A AAAA

"There's nothing between aaa and zzz"
Therefore nonexistent.example.com doesn't exist.

NSEC3: Hashed version (prevents zone enumeration)
```

## Chain of Trust

DNSSEC builds a chain from root to leaf:

```
                    ┌──────────────────┐
                    │   Root Zone (.)  │
                    │                  │
                    │  DNSKEY (root)   │ ← Hardcoded in resolvers
                    └────────┬─────────┘   (trust anchor)
                             │
                    Signed DS record for .com
                             │
                    ┌────────▼─────────┐
                    │   .com TLD       │
                    │                  │
                    │  DNSKEY (.com)   │
                    └────────┬─────────┘
                             │
                    Signed DS record for example.com
                             │
                    ┌────────▼─────────┐
                    │   example.com    │
                    │                  │
                    │  DNSKEY          │
                    │  A record + RRSIG│
                    └──────────────────┘

Each level signs the next level's key hash.
Trust flows from root anchor to leaf records.
```

## Validation Process

```
Resolver validating example.com A record:

1. Get example.com A + RRSIG
2. Get example.com DNSKEY
3. Verify RRSIG with DNSKEY ✓

4. Get DS for example.com (from .com zone)
5. Verify DS matches DNSKEY hash ✓

6. Get .com DNSKEY
7. Verify DS RRSIG with .com DNSKEY ✓

8. Get DS for .com (from root zone)
9. Verify DS matches .com DNSKEY hash ✓

10. Get root DNSKEY
11. Verify against trust anchor ✓

All checks pass → Record is authenticated!
Any check fails → SERVFAIL (reject response)
```

## Querying DNSSEC Records

```bash
# Request DNSSEC records
$ dig example.com +dnssec

# Check if domain is signed
$ dig example.com DNSKEY
$ dig example.com DS

# Verify signature chain
$ dig +sigchase example.com

# Use delv (DNSSEC-aware dig)
$ delv example.com
; fully validated
example.com.    86400    IN    A    93.184.216.34

# Check validation status
$ dig +cd example.com    # CD = Checking Disabled (skip validation)
```

## DNSSEC Status Check

```bash
# Online validators:
# https://dnssec-analyzer.verisignlabs.com/
# https://dnsviz.net/

# Command line check:
$ delv @8.8.8.8 example.com
; fully validated         ← DNSSEC working
; unsigned answer         ← Not signed
; validation failed       ← Signature invalid

# Check with drill
$ drill -S example.com
```

## Key Management

### Key Types

```
Zone Signing Key (ZSK):
  - Signs zone records
  - Rotated frequently (monthly to quarterly)
  - Smaller key (faster signing)

Key Signing Key (KSK):
  - Signs the ZSK
  - Rotated less often (yearly)
  - Referenced by parent's DS record
  - Larger key (more security)

Why two keys?
  ZSK rotation doesn't require parent update
  KSK rotation requires new DS in parent zone
```

### Key Rollover

```
ZSK Rollover (simpler):
  1. Generate new ZSK
  2. Publish both old and new DNSKEY
  3. Sign with new ZSK
  4. After TTL, remove old ZSK

KSK Rollover (complex):
  1. Generate new KSK
  2. Publish both DNSKEYs
  3. Submit new DS to parent
  4. Wait for parent propagation
  5. Sign ZSKs with new KSK
  6. After parent DS TTL, remove old KSK

Automated by most DNS providers.
```

## Deployment Considerations

### Enabling DNSSEC

```
Domain owner must:
  1. Sign zone with DNSSEC keys
  2. Upload DS record to registrar
  3. Registrar submits DS to TLD
  4. Chain of trust established

Many registrars/DNS providers automate this:
  - Cloudflare: One-click DNSSEC
  - Route53: Supports DNSSEC
  - Google Domains: Easy setup
```

### Response Size

```
DNSSEC adds significant size:

Without DNSSEC:
  example.com A → ~50 bytes

With DNSSEC:
  example.com A + RRSIG + DNSKEY → ~1000+ bytes

Implications:
  - May exceed 512-byte UDP limit
  - Requires EDNS (larger UDP) or TCP
  - More bandwidth usage
```

### Validation Failures

```
If DNSSEC validation fails:

Validating resolver returns: SERVFAIL
User sees: DNS error / site unreachable

Causes:
  - Expired signatures (operator forgot renewal)
  - Incorrect DS record (misconfiguration)
  - Clock skew (signature timestamps)
  - Key rollover problems

This is a feature, not a bug!
Invalid signatures could mean attack.
But operational errors can cause outages.
```

## Limitations

DNSSEC protects authenticity, not privacy:

```
DNSSEC provides:
  ✓ Authentication (record from legitimate source)
  ✓ Integrity (record not modified)
  ✓ Authenticated denial (NXDOMAIN is real)

DNSSEC does NOT provide:
  ✗ Confidentiality (queries/responses visible)
  ✗ Protection from DNS operator
  ✗ Protection of last-mile (resolver to client)

For privacy: DNS over HTTPS (DoH) or DNS over TLS (DoT)
```

## DNSSEC Adoption

```
Adoption varies by TLD:

Signed TLDs: .com, .org, .net (all major TLDs)

Domain signing rates:
  .nl (Netherlands):  ~50%
  .se (Sweden):       ~40%
  .com:               ~3%

Validation by resolvers:
  8.8.8.8 (Google):     Validates
  1.1.1.1 (Cloudflare): Validates
  ISP resolvers:        Varies

Growing but not universal.
```

## Alternatives and Complements

### DNS over HTTPS (DoH)

```
HTTPS encryption for DNS queries:
  - Hides queries from network observers
  - Bypasses some filtering
  - Runs on port 443 (like web traffic)

Complements DNSSEC:
  DoH   = Privacy (encrypted transport)
  DNSSEC = Authenticity (signed records)

Can use both together.
```

### DNS over TLS (DoT)

```
TLS encryption for DNS:
  - Dedicated port 853
  - Easier to identify/block than DoH
  - Same privacy benefits as DoH

Adoption growing in mobile and resolvers.
```

## Summary

DNSSEC adds cryptographic security to DNS:

| Component | Purpose |
|-----------|---------|
| DNSKEY | Zone's public keys |
| RRSIG | Signatures on record sets |
| DS | Links child to parent (trust chain) |
| NSEC/NSEC3 | Proves non-existence |

Key points:
- **Chain of trust** from root to leaf
- **Signatures** prevent tampering
- **Validation failures** block responses
- **Doesn't provide privacy** (use DoH/DoT)

Considerations:
- Operational complexity (key management)
- Larger responses (more bandwidth)
- Validation failures can cause outages
- Growing but not universal adoption

This completes our DNS coverage. Next, we'll explore the evolution of HTTP—from 1.0 to HTTP/3.
