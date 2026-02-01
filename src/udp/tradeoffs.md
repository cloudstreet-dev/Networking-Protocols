# UDP vs TCP Trade-offs

Choosing between UDP and TCP isn't about which is "better"—it's about understanding the trade-offs and matching them to your requirements. This chapter provides a detailed comparison to help you make informed decisions.

## Feature Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│               Feature                  │    TCP    │    UDP    │        │
├────────────────────────────────────────┼───────────┼───────────┼────────┤
│ Reliable delivery                      │    ✓      │    ✗      │        │
│ Ordered delivery                       │    ✓      │    ✗      │        │
│ Error detection                        │    ✓      │    ✓*     │ *opt.  │
│ Flow control                           │    ✓      │    ✗      │        │
│ Congestion control                     │    ✓      │    ✗      │        │
│ Connection-oriented                    │    ✓      │    ✗      │        │
│ Message boundaries                     │    ✗      │    ✓      │        │
│ Broadcast/Multicast                    │    ✗      │    ✓      │        │
│ NAT traversal friendly                 │    ✓      │   varies  │        │
│ Firewall friendly                      │    ✓      │    ✗      │        │
└────────────────────────────────────────┴───────────┴───────────┴────────┘
```

## Latency Analysis

### Connection Establishment

```
TCP - New Connection:
┌────────────────────────────────────────────────────────────┐
│  0ms    Client sends SYN                                   │
│  50ms   Server receives, sends SYN-ACK                     │
│  100ms  Client receives, sends ACK + first data            │
│  150ms  Server receives first data                         │
│                                                            │
│  Minimum latency to first data: 1.5 RTT                    │
└────────────────────────────────────────────────────────────┘

TCP - Established Connection:
┌────────────────────────────────────────────────────────────┐
│  0ms    Client sends data                                  │
│  50ms   Server receives data                               │
│                                                            │
│  Latency: 0.5 RTT (one-way)                                │
└────────────────────────────────────────────────────────────┘

UDP:
┌────────────────────────────────────────────────────────────┐
│  0ms    Client sends datagram                              │
│  50ms   Server receives datagram                           │
│                                                            │
│  Latency: 0.5 RTT (always)                                 │
│  No connection overhead!                                   │
└────────────────────────────────────────────────────────────┘
```

### Request-Response Latency

```
Single request, single response:

TCP (new connection):
  Handshake:  1 RTT
  Request:    0.5 RTT
  Response:   0.5 RTT
  Total:      2 RTT

TCP (existing connection):
  Request:    0.5 RTT
  Response:   0.5 RTT
  Total:      1 RTT

UDP:
  Request:    0.5 RTT
  Response:   0.5 RTT
  Total:      1 RTT

For one-shot interactions, UDP saves 1 RTT.
For repeated interactions, TCP connection reuse matches UDP.
```

### Latency Under Loss

```
5% packet loss scenario:

TCP:
  Packet lost → Detected (3 dup ACKs or timeout)
  Fast retransmit: ~1 RTT additional delay
  Timeout: Several seconds delay!

  Also: Congestion window reduced
        Subsequent packets slowed

UDP:
  Packet lost → Application decides:
    - Ignore it (real-time)
    - Request retransmit (application-level)
    - Interpolate from adjacent data

  No cascading effects on other packets.
```

## Throughput Analysis

### Header Overhead

```
Per-packet overhead:

TCP: 20-60 bytes (typically 32 with timestamps)
UDP: 8 bytes

Efficiency for 100-byte payload:
  TCP: 100 / 132 = 76%
  UDP: 100 / 108 = 93%

Efficiency for 1400-byte payload:
  TCP: 1400 / 1432 = 98%
  UDP: 1400 / 1408 = 99%

UDP's advantage shrinks with larger payloads.
Matters most for small messages.
```

### Maximum Throughput

```
TCP:
  Limited by: min(cwnd, rwnd) / RTT
  Congestion control prevents network overload
  Fair sharing with other flows

  Example: 64KB window, 50ms RTT
           Max: 64KB / 50ms = 1.28 MB/s

UDP:
  Limited by: Application send rate
  No built-in limits!
  Can overwhelm network

  Can achieve wire speed... if network allows.
  But may cause massive loss and collateral damage.
```

### Behavior Under Congestion

```
Network congested:

TCP:
  Detects loss → Reduces cwnd
  Backs off → Congestion clears
  Gradually increases again
  "Good citizen" - shares fairly

UDP:
  No awareness of congestion
  Keeps sending at same rate
  Causes more congestion
  Other TCP flows suffer

This is why uncontrolled UDP can harm the network.
Responsible UDP apps implement their own congestion control.
```

## Reliability Implications

### Handling Loss

```
TCP handles loss automatically:
  1. Detects via ACK timeout or dup ACKs
  2. Retransmits lost segment
  3. Adjusts congestion window
  4. Application sees reliable byte stream

UDP loss is application's problem:
  1. Application must detect (if it cares)
  2. Application must request retransmit (if needed)
  3. Application decides what to do

Sometimes that's a feature:
  Video codec can mask lost frame
  Game can interpolate missing position
  Voice can use error concealment
```

### Ordering Implications

```
TCP guarantees order:
  Sent: A B C D E
  Received: A B C D E (always)

  If C is lost:
    B arrives, delivered
    D arrives, buffered
    E arrives, buffered
    C retransmitted, arrives
    C D E delivered in order

UDP makes no guarantee:
  Sent: A B C D E
  Received: A B D C E (possible)
            A B D E (C lost)
            A D B C E (reordered)

Application must handle or ignore.
```

## Resource Usage

### Server Memory

```
TCP Server (10,000 connections):
  Per connection:
    - Socket structure
    - Send buffer (~16KB)
    - Receive buffer (~16KB)
    - TCP control block

  Total: ~320MB for buffers alone
         Plus connection tracking overhead

UDP Server (10,000 "clients"):
  Single socket:
    - One send buffer
    - One receive buffer
    - No connection state!

  Total: ~32KB
         Applications track state if needed

UDP scales better for many ephemeral interactions.
```

### CPU Usage

```
TCP per packet:
  - Checksum calculation
  - Sequence number tracking
  - ACK generation
  - Window management
  - Congestion control
  - Timer management

UDP per packet:
  - Checksum calculation (optional in IPv4)
  - That's it

UDP has lower CPU overhead per packet.
But if you implement reliability, you add CPU work.
```

## NAT and Firewall Behavior

### NAT Traversal

```
TCP through NAT:
  1. Client connects out
  2. NAT creates mapping
  3. Server responses follow mapping
  4. Works reliably

UDP through NAT:
  1. Client sends datagram out
  2. NAT creates mapping
  3. Mapping may timeout quickly!
  4. Need keepalive packets

UDP NAT mappings often timeout in 30-120 seconds.
Long-lived UDP "connections" need periodic keepalive.
```

### Firewall Policies

```
Common firewall behavior:

Corporate firewalls:
  TCP 80 (HTTP): Usually allowed
  TCP 443 (HTTPS): Usually allowed
  UDP 53 (DNS): Often allowed
  UDP 123 (NTP): Sometimes allowed
  Other UDP: Often blocked!

If targeting corporate networks:
  UDP may not work
  TCP or WebSocket more reliable
  HTTPS most reliable
```

## When to Choose What

### Choose TCP When:

```
✓ Data integrity critical (files, transactions)
✓ Simple implementation preferred
✓ Operating through corporate firewalls
✓ Long-lived connections
✓ Need reliable delivery without custom code
✓ Building on HTTP, TLS, or other TCP protocols
```

### Choose UDP When:

```
✓ Real-time requirements (voice, video, gaming)
✓ Broadcast or multicast needed
✓ Small, independent messages
✓ Custom reliability acceptable
✓ Willing to implement congestion control
✓ Protocol requires it (DNS, DHCP, QUIC)
```

### Consider QUIC When:

```
✓ Want UDP benefits with reliability
✓ Need multiple streams without HoL blocking
✓ Want 0-RTT connection resumption
✓ Willing to use a more complex library
✓ Building modern web services
```

## Performance Comparison Summary

```
┌────────────────────────────────────────────────────────────────────────┐
│ Metric                   │ TCP                 │ UDP                   │
├──────────────────────────┼─────────────────────┼───────────────────────┤
│ Initial latency          │ 1-1.5 RTT overhead  │ No overhead           │
│ Steady-state latency     │ Similar             │ Similar               │
│ Latency under loss       │ High (retransmit)   │ Low (skip if desired) │
│ Throughput (clean)       │ Good                │ Can exceed            │
│ Throughput (lossy)       │ Degrades gracefully │ Application-dependent │
│ Header overhead          │ 20-60 bytes         │ 8 bytes               │
│ Server memory            │ High                │ Low                   │
│ Server CPU               │ Moderate            │ Low                   │
│ Implementation effort    │ Low (OS handles)    │ High (if reliability) │
└────────────────────────────────────────────────┴───────────────────────┘
```

## Hybrid Approaches

Many applications use both:

```
Example: Online Game

TCP for:
  - Authentication
  - Chat messages
  - Purchases/transactions
  - Downloading updates

UDP for:
  - Player positions
  - World state
  - Audio chat
  - Time-sensitive events

Single codebase, two transports, best of both worlds.
```

## Summary

The choice between TCP and UDP depends on your specific requirements:

| Requirement | Prefer |
|-------------|--------|
| Simplicity | TCP |
| Reliability built-in | TCP |
| Lowest latency | UDP |
| Real-time tolerance for loss | UDP |
| Broadcast/multicast | UDP |
| Corporate firewall traversal | TCP |
| Custom protocol over UDP | Consider QUIC |

Neither protocol is universally "better." Understanding the trade-offs lets you make the right choice for your application—or use both where appropriate.

This completes our coverage of UDP. Next, we'll explore DNS—the internet's naming system that typically uses UDP for queries.
