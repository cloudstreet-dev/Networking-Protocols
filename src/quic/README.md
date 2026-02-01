# QUIC Protocol

**QUIC** is a general-purpose transport protocol that originated at Google and is now standardized by the IETF. While HTTP/3 is its most visible use, QUIC can transport any application protocol that currently uses TCP.

## QUIC at a Glance

```
┌─────────────────────────────────────────────────────────────────────┐
│                        QUIC Features                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Transport Layer                                                    │
│    ✓ Reliable, ordered delivery per stream                         │
│    ✓ Congestion control                                             │
│    ✓ Flow control (connection and stream level)                     │
│                                                                     │
│  Encryption                                                         │
│    ✓ TLS 1.3 integrated (mandatory)                                 │
│    ✓ Encrypted headers and payload                                  │
│    ✓ Protected from middlebox interference                          │
│                                                                     │
│  Multiplexing                                                       │
│    ✓ Independent streams (no HOL blocking)                          │
│    ✓ Bidirectional and unidirectional streams                       │
│    ✓ Stream priorities                                              │
│                                                                     │
│  Connection                                                         │
│    ✓ Connection IDs (survives IP changes)                           │
│    ✓ 0-RTT resumption                                               │
│    ✓ 1-RTT initial connection                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Why Build on UDP?

```
Why not improve TCP?

1. Kernel Dependency
   TCP is implemented in OS kernels
   Changes require kernel updates
   Deployment takes years

2. Middlebox Ossification
   Firewalls, NATs inspect TCP headers
   "Unknown" TCP options get dropped
   TCP extensions rarely deploy successfully

3. Head-of-Line Blocking
   TCP's byte stream model is fundamental
   Cannot fix without breaking compatibility

QUIC on UDP:
   - Implemented in userspace (fast iteration)
   - UDP passes through middleboxes unchanged
   - Full control over protocol behavior
   - Can add features without kernel changes
```

## What You'll Learn

This chapter covers:

1. **Why QUIC Exists**: The problems it solves
2. **Connection Establishment**: 0-RTT and 1-RTT handshakes
3. **Multiplexing**: How streams eliminate HOL blocking
4. **Connection Migration**: Surviving network changes

QUIC is the future of transport protocols. Understanding it prepares you for where networking is heading.
