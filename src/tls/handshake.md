# The TLS Handshake

The TLS handshake establishes a secure connection by negotiating cryptographic parameters and authenticating the server. Understanding it helps debug connection issues and appreciate TLS 1.3's improvements.

## TLS 1.2 Handshake

```
Client                                               Server
   │                                                    │
   │─────────── ClientHello ───────────────────────────>│
   │  - TLS version                                     │
   │  - Random bytes                                    │
   │  - Cipher suites supported                         │
   │  - Extensions (SNI, etc.)                          │
   │                                                    │
   │<────────── ServerHello ────────────────────────────│
   │  - Chosen TLS version                              │
   │  - Server random                                   │
   │  - Selected cipher suite                           │
   │                                                    │
   │<────────── Certificate ────────────────────────────│
   │  - Server's certificate chain                      │
   │                                                    │
   │<────────── ServerKeyExchange ─────────────────────│
   │  - Key exchange parameters (if needed)             │
   │                                                    │
   │<────────── ServerHelloDone ───────────────────────│
   │                                                    │
   │─────────── ClientKeyExchange ─────────────────────>│
   │  - Pre-master secret (encrypted)                   │
   │                                                    │
   │─────────── ChangeCipherSpec ──────────────────────>│
   │  - "Switching to encrypted"                        │
   │                                                    │
   │─────────── Finished ──────────────────────────────>│
   │  - Encrypted verification                          │
   │                                                    │
   │<────────── ChangeCipherSpec ───────────────────────│
   │<────────── Finished ───────────────────────────────│
   │                                                    │
   │══════════ Encrypted Application Data ══════════════│
```

**TLS 1.2: 2 round trips before application data**

## TLS 1.3 Handshake (Simplified)

```
Client                                               Server
   │                                                    │
   │─────────── ClientHello ───────────────────────────>│
   │  - TLS 1.3                                         │
   │  - Supported groups & key shares                   │
   │  - Signature algorithms                            │
   │                                                    │
   │<────────── ServerHello ────────────────────────────│
   │  - Selected key share                              │
   │                                                    │
   │<────────── EncryptedExtensions ────────────────────│
   │<────────── Certificate ────────────────────────────│
   │<────────── CertificateVerify ─────────────────────│
   │<────────── Finished ───────────────────────────────│
   │                                                    │
   │─────────── Finished ──────────────────────────────>│
   │                                                    │
   │══════════ Encrypted Application Data ══════════════│
```

**TLS 1.3: 1 round trip before application data**

## Key Exchange

How do client and server agree on encryption keys?

### Diffie-Hellman Key Exchange

```
The magic: Agree on a shared secret over an insecure channel.

1. Public parameters: Prime p, Generator g

2. Client picks random a, computes A = g^a mod p
   Server picks random b, computes B = g^b mod p

3. Exchange A and B (visible to eavesdroppers)

4. Client computes: secret = B^a mod p = g^(ab) mod p
   Server computes: secret = A^b mod p = g^(ab) mod p

Both have same secret! Eavesdropper has A and B but
cannot compute g^(ab) without knowing a or b (discrete log problem).

Modern TLS uses Elliptic Curve Diffie-Hellman (ECDHE) for efficiency.
```

### Perfect Forward Secrecy

```
Why ephemeral keys matter:

Without PFS (RSA key exchange):
  - Server's long-term key encrypts pre-master secret
  - If key later compromised, all past traffic decryptable

With PFS (ECDHE):
  - New DH keys generated per session
  - Session keys destroyed after use
  - Compromising server key doesn't reveal past sessions

TLS 1.3 requires PFS (ECDHE or DHE only).
```

## Server Name Indication (SNI)

```
Problem: Single IP hosts multiple HTTPS sites.
         Which certificate should server present?

Solution: SNI extension in ClientHello.

ClientHello includes:
  server_name = "www.example.com"

Server sees hostname BEFORE certificate selection.
Presents correct certificate for that hostname.

Note: SNI is sent unencrypted in TLS 1.2.
      Encrypted Client Hello (ECH) in TLS 1.3 hides it.
```

## Session Resumption

Avoiding full handshake for repeat connections:

### Session IDs (TLS 1.2)

```
First connection: Full handshake, server assigns session ID
Subsequent: Client presents session ID, server looks up keys
            Abbreviated handshake (1 RTT instead of 2)

Limitation: Server must store session state (doesn't scale).
```

### Session Tickets (TLS 1.2)

```
Server encrypts session state into ticket.
Client stores ticket, presents on reconnection.
Server decrypts ticket, recovers session state.

Advantage: Stateless server, better scaling.
```

### 0-RTT Resumption (TLS 1.3)

```
Client sends:
  ClientHello + Early Data (encrypted with previous session key)

Server can respond to early data immediately.
No round trip before application data!

Security caveat: Early data is replayable.
Only safe for idempotent requests.
```

## Handshake Failures

### Certificate Errors

```
ERR_CERT_AUTHORITY_INVALID
  - Certificate not trusted
  - Self-signed or unknown CA
  - Missing intermediate certificate

ERR_CERT_DATE_INVALID
  - Certificate expired
  - Certificate not yet valid
  - System clock wrong

ERR_CERT_COMMON_NAME_INVALID
  - Hostname doesn't match certificate
  - Wrong server or misconfiguration
```

### Protocol Errors

```
ERR_SSL_VERSION_OR_CIPHER_MISMATCH
  - No common TLS version
  - No common cipher suite
  - Often: Server only supports old protocols

ERR_SSL_PROTOCOL_ERROR
  - Malformed handshake messages
  - Middlebox interference
  - Implementation bugs
```

## Debugging TLS

```bash
# OpenSSL client
$ openssl s_client -connect example.com:443 -servername example.com

# Show certificate
$ openssl s_client -connect example.com:443 2>/dev/null | \
    openssl x509 -text -noout

# Test specific TLS version
$ openssl s_client -connect example.com:443 -tls1_2
$ openssl s_client -connect example.com:443 -tls1_3

# curl with verbose TLS info
$ curl -v https://example.com 2>&1 | grep -i ssl

# Test suite (ssllabs.com/ssltest online, or testssl.sh locally)
$ ./testssl.sh example.com
```

## Summary

TLS handshake accomplishes:

| Goal | Mechanism |
|------|-----------|
| Version negotiation | ClientHello/ServerHello |
| Cipher suite selection | ClientHello/ServerHello |
| Server authentication | Certificate + signature |
| Key exchange | ECDHE (Diffie-Hellman) |
| Forward secrecy | Ephemeral keys |
| Session resumption | Tickets, 0-RTT |

TLS 1.3 improvements:
- 1-RTT handshake (vs 2-RTT)
- 0-RTT resumption option
- Only secure cipher suites
- Encrypted handshake data
- Simpler, more secure
