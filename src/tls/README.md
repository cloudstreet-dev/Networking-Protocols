# TLS/SSL

**TLS (Transport Layer Security)** encrypts network communication, protecting data from eavesdropping and tampering. It's what puts the "S" in HTTPS and secures most internet traffic today.

## What TLS Provides

```
┌─────────────────────────────────────────────────────────────────────┐
│                       TLS Security Goals                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Confidentiality                                                    │
│    Data encrypted, only endpoints can read it                       │
│    Eavesdropper sees random bytes                                   │
│                                                                     │
│  Integrity                                                          │
│    Data tampering detected                                          │
│    HMAC ensures message authenticity                                │
│                                                                     │
│  Authentication                                                     │
│    Server proves identity via certificate                           │
│    Client verifies it's talking to real server                      │
│    (Optional: Client can prove identity too)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## TLS in the Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Protocol Stack                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────┐          │
│  │              Application (HTTP, SMTP, etc.)           │          │
│  ├───────────────────────────────────────────────────────┤          │
│  │                        TLS                            │ ← Here   │
│  ├───────────────────────────────────────────────────────┤          │
│  │                        TCP                            │          │
│  ├───────────────────────────────────────────────────────┤          │
│  │                        IP                             │          │
│  └───────────────────────────────────────────────────────┘          │
│                                                                     │
│  TLS sits between application and transport.                        │
│  Application sees plain data.                                       │
│  Network sees encrypted data.                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Brief History

```
1995: SSL 2.0 (Netscape) - First public version, insecure
1996: SSL 3.0 - Major improvements, still vulnerabilities
1999: TLS 1.0 - Standardized by IETF, based on SSL 3.0
2006: TLS 1.1 - Minor security improvements
2008: TLS 1.2 - Modern cipher suites, still widely used
2018: TLS 1.3 - Simplified, faster, more secure

Today:
  TLS 1.3 preferred
  TLS 1.2 acceptable
  TLS 1.0/1.1 deprecated (should be disabled)
  SSL: Do not use
```

## How TLS Works (Overview)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TLS Connection                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Handshake                                                       │
│     - Client and server negotiate parameters                        │
│     - Server presents certificate                                   │
│     - Key exchange establishes shared secret                        │
│     - Derive session keys                                           │
│                                                                     │
│  2. Encrypted Communication                                         │
│     - All data encrypted with session keys                          │
│     - MAC ensures integrity                                         │
│     - Sequence numbers prevent replay                               │
│                                                                     │
│  3. Closure                                                         │
│     - Close notify alert                                            │
│     - Prevents truncation attacks                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## What You'll Learn

1. **The TLS Handshake**: How secure connections are established
2. **Certificates and PKI**: How identity is verified
3. **Cipher Suites**: The cryptographic algorithms used
4. **TLS 1.3 Improvements**: What makes the latest version better
