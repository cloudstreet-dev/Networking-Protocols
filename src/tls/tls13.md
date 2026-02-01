# TLS 1.3 Improvements

TLS 1.3 (RFC 8446, 2018) represents a major overhaul, not just incremental improvement. It's faster, simpler, and more secure than TLS 1.2.

## Key Improvements

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TLS 1.3 vs TLS 1.2                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Performance:                                                       │
│    TLS 1.2: 2 RTT handshake                                         │
│    TLS 1.3: 1 RTT handshake                                         │
│             0 RTT resumption                                        │
│                                                                     │
│  Security:                                                          │
│    Removed: RSA key exchange, CBC ciphers, SHA-1, RC4, 3DES         │
│    Required: Forward secrecy (ECDHE/DHE only)                       │
│    Encrypted: More handshake data hidden                            │
│                                                                     │
│  Simplicity:                                                        │
│    Cipher suites: 37+ → 5                                           │
│    Fewer options = fewer misconfigurations                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 1-RTT Handshake

TLS 1.3 combines key exchange and authentication:

```
TLS 1.2:
  ClientHello        →            1st RTT
  ←  ServerHello + Cert
  ClientKeyExchange  →            2nd RTT
  ←  Finished
  Application Data   →            Finally!

TLS 1.3:
  ClientHello + KeyShare  →       1st RTT
  ←  ServerHello + KeyShare
  ←  EncryptedExtensions
  ←  Certificate, Finished
  Finished                →
  Application Data        →       Immediately!

Client sends key share in first message.
Server can compute keys immediately.
Encrypted data flows after 1 RTT.
```

## 0-RTT Resumption

For returning clients:

```
TLS 1.3 0-RTT:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  First Connection:                                                  │
│    Full handshake + receive session ticket                          │
│                                                                     │
│  Subsequent Connection:                                             │
│    ClientHello + EarlyData  →  (request sent IMMEDIATELY)           │
│    ←  ServerHello + response                                        │
│                                                                     │
│  No waiting! Request in first packet.                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Security caveat:
  0-RTT data can be replayed by attacker
  Only safe for idempotent operations (GET, not POST)
  Server can reject 0-RTT for sensitive operations
```

## Removed Features

TLS 1.3 removed insecure and unnecessary features:

```
Removed entirely:
  ✗ RSA key exchange (no forward secrecy)
  ✗ Static Diffie-Hellman (no forward secrecy)
  ✗ CBC mode ciphers (padding oracle attacks)
  ✗ RC4 (broken)
  ✗ 3DES (slow, small block size)
  ✗ MD5 and SHA-1 in signature algorithms
  ✗ Compression (CRIME attack)
  ✗ Renegotiation
  ✗ Custom DHE groups
  ✗ ChangeCipherSpec message

Result: All TLS 1.3 connections have forward secrecy
        and use authenticated encryption (AEAD).
```

## Encrypted Handshake

More of the handshake is encrypted:

```
TLS 1.2 visible to eavesdropper:
  - Certificate (server identity)
  - Server extensions
  - Much of handshake

TLS 1.3 encrypted:
  - Certificate
  - Extensions after ServerHello
  - Most handshake messages

Only visible:
  - ClientHello (including SNI)
  - ServerHello

Future: Encrypted Client Hello (ECH) hides SNI too.
```

## Simplified Cipher Suites

```
TLS 1.2: 37+ cipher suites (many weak/redundant)
TLS 1.3: 5 cipher suites (all secure)

TLS 1.3 cipher suites:
  TLS_AES_128_GCM_SHA256          (required)
  TLS_AES_256_GCM_SHA384          (recommended)
  TLS_CHACHA20_POLY1305_SHA256    (good for non-AES hardware)
  TLS_AES_128_CCM_SHA256          (IoT)
  TLS_AES_128_CCM_8_SHA256        (IoT, constrained)

Key exchange negotiated separately via supported_groups.
Signature algorithms negotiated separately.
Simpler configuration, fewer mistakes.
```

## Downgrade Protection

TLS 1.3 prevents protocol downgrade attacks:

```
Attack scenario:
  Client supports TLS 1.3
  Server supports TLS 1.3
  Attacker modifies ClientHello to say "TLS 1.2 only"
  Connection uses weaker TLS 1.2

Protection:
  Server random includes special bytes when downgrading
  Client detects this and aborts
  Man-in-the-middle cannot force downgrade
```

## Migration Considerations

### Compatibility

```
TLS 1.3 designed for compatibility:
  - Uses same port (443)
  - Can negotiate down to TLS 1.2 if needed
  - Works with most proxies/load balancers

Potential issues:
  - Old middleboxes may break 1.3
  - Some intrusion detection fails on 1.3
  - 0-RTT requires application awareness
```

### Server Configuration

```nginx
# nginx - enable TLS 1.3
ssl_protocols TLSv1.2 TLSv1.3;

# Enable 0-RTT (use with caution)
ssl_early_data on;

# In proxy situations, tell backend about early data
proxy_set_header Early-Data $ssl_early_data;
```

### Application Changes for 0-RTT

```python
# Check if request was 0-RTT
early_data = request.headers.get('Early-Data')

if early_data == '1':
    # This request might be replayed!
    if not is_idempotent(request):
        # Reject or require retry without 0-RTT
        return Response(status=425)  # Too Early
```

## Measuring TLS 1.3 Adoption

```
As of 2024:
  - ~70% of websites support TLS 1.3
  - All major browsers support TLS 1.3
  - All major CDNs support TLS 1.3

Verify your site:
  $ curl -v https://yoursite.com 2>&1 | grep "SSL connection"
  * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

## Summary

TLS 1.3 advantages:

| Feature | Improvement |
|---------|-------------|
| Handshake | 1 RTT (vs 2 RTT) |
| Resumption | 0 RTT possible |
| Security | Only secure options remain |
| Configuration | 5 ciphers vs 37+ |
| Privacy | More encrypted handshake |
| Forward secrecy | Mandatory |

TLS 1.3 should be enabled on all new deployments. The only reason to stay on TLS 1.2 is legacy client compatibility, and that's decreasing rapidly.
