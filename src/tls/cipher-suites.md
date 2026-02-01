# Cipher Suites

A **cipher suite** is a combination of cryptographic algorithms used in a TLS connection. Understanding them helps you configure secure connections and debug compatibility issues.

## Cipher Suite Components

```
TLS 1.2 cipher suite name:
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  │   │     │       │   │   │   │
  │   │     │       │   │   │   └── PRF hash
  │   │     │       │   │   └────── Mode (GCM)
  │   │     │       │   └────────── Key size (256-bit)
  │   │     │       └────────────── Encryption (AES)
  │   │     └────────────────────── Authentication (RSA cert)
  │   └──────────────────────────── Key Exchange (ECDHE)
  └──────────────────────────────── Protocol

Components:
  Key Exchange: How to agree on encryption keys
  Authentication: How to verify server identity
  Encryption: How to encrypt data
  MAC/Hash: How to verify integrity
```

## Algorithm Categories

### Key Exchange

```
RSA           Server's RSA key encrypts pre-master secret
              No forward secrecy
              DO NOT USE (TLS 1.3 removed)

DHE           Diffie-Hellman Ephemeral
              Forward secrecy
              Slower than ECDHE

ECDHE         Elliptic Curve Diffie-Hellman Ephemeral
              Forward secrecy
              Fast, secure
              RECOMMENDED
```

### Authentication

```
RSA           RSA certificate, RSA signature
              Widely supported

ECDSA         Elliptic Curve DSA
              Smaller keys, faster
              Growing adoption

EdDSA         Ed25519/Ed448
              Modern, fast
              TLS 1.3 support
```

### Bulk Encryption

```
AES-GCM       AES Galois/Counter Mode (AEAD)
              Fast, secure, authenticated encryption
              RECOMMENDED

AES-CBC       AES Cipher Block Chaining
              Older, requires separate MAC
              Vulnerable to padding oracles
              AVOID

ChaCha20-Poly1305  Stream cipher (AEAD)
              Fast on devices without AES hardware
              Good alternative to AES-GCM
```

### MAC/Hash

```
SHA-384       For AEAD ciphers, used in PRF
SHA-256       For AEAD ciphers, used in PRF

SHA-1         Old, deprecated
              Only for compatibility
              DO NOT USE if avoidable
```

## TLS 1.2 Recommended Suites

```
Preferred (forward secrecy, AEAD):
  TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
  TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256

Acceptable (for compatibility):
  TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
  TLS_DHE_RSA_WITH_AES_128_GCM_SHA256

Avoid:
  Anything with RSA key exchange (no PFS)
  Anything with CBC mode (padding attacks)
  Anything with 3DES (slow, weak)
  Anything with RC4 (broken)
  Anything with NULL (no encryption!)
  Anything with EXPORT (intentionally weak)
```

## TLS 1.3 Cipher Suites

TLS 1.3 simplified cipher suites dramatically:

```
Only 5 cipher suites:
  TLS_AES_256_GCM_SHA384
  TLS_AES_128_GCM_SHA256
  TLS_CHACHA20_POLY1305_SHA256
  TLS_AES_128_CCM_SHA256
  TLS_AES_128_CCM_8_SHA256

Key exchange (ECDHE) and authentication (certificate signature)
are negotiated separately via extensions.

All TLS 1.3 suites provide:
  ✓ Forward secrecy (mandatory)
  ✓ AEAD encryption (mandatory)
  ✓ Strong algorithms (weak ones removed)
```

## Configuring Cipher Suites

### nginx

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_cipher_on;

# TLS 1.2 ciphers
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

# TLS 1.3 ciphers (usually automatic)
ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
```

### Apache

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:...
SSLHonorCipherOrder on
```

## Testing Cipher Suites

```bash
# Test supported ciphers
$ nmap --script ssl-enum-ciphers -p 443 example.com

# OpenSSL test specific cipher
$ openssl s_client -connect example.com:443 \
    -cipher ECDHE-RSA-AES256-GCM-SHA384

# Show negotiated cipher
$ openssl s_client -connect example.com:443 2>/dev/null | \
    grep "Cipher is"
# New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384

# SSL Labs test (comprehensive)
# https://www.ssllabs.com/ssltest/
```

## Security Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│  Level    │ Key Exchange  │ Encryption    │ Bits of Security       │
├───────────┼───────────────┼───────────────┼────────────────────────┤
│  Modern   │ ECDHE P-256+  │ AES-128-GCM+  │ 128-bit security       │
│           │               │ ChaCha20      │                        │
├───────────┼───────────────┼───────────────┼────────────────────────┤
│  Compat.  │ ECDHE/DHE     │ AES-128+      │ 112-128 bit            │
│           │ 2048-bit+     │               │                        │
├───────────┼───────────────┼───────────────┼────────────────────────┤
│  Legacy   │ RSA 2048      │ AES/3DES      │ ~80-112 bit            │
│  (avoid)  │               │               │                        │
├───────────┼───────────────┼───────────────┼────────────────────────┤
│  Broken   │ RSA < 2048    │ RC4, DES      │ Effectively none       │
│  (never)  │ EXPORT        │ NULL          │                        │
└───────────┴───────────────┴───────────────┴────────────────────────┘
```

## Summary

Good cipher suite configuration:

1. **Use TLS 1.3** when possible (automatic good choices)
2. **Prefer ECDHE** for key exchange (forward secrecy)
3. **Use AEAD** encryption (AES-GCM or ChaCha20-Poly1305)
4. **Disable weak ciphers** (RSA key exchange, CBC, old algorithms)
5. **Test your configuration** (SSL Labs, testssl.sh)

TLS 1.3 removes the complexity—all its cipher suites are secure.
